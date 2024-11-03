# 快速消息队列 (FMQ)

bookmark_border



HIDL 的远程过程调用 (RPC) 基础设施使用 Binder 机制，这意味着调用涉及开销，需要内核操作，并且可能触发调度程序操作。但是，对于必须在开销较小且不涉及内核的进程之间传输数据的情况，可以使用快速消息队列 (FMQ) 系统。

FMQ 创建具有所需属性的消息队列。 `MQDescriptorSync`或`MQDescriptorUnsync`对象可以通过 HIDL RPC 调用发送，并由接收进程用于访问消息队列。

快速消息队列仅在 C++ 和运行 Android 8.0 及更高版本的设备上受支持。

## 消息队列类型

Android 支持两种队列类型（称为*flavors* ）：

- *非同步*队列允许溢出，可以有很多读者；每个读者必须及时读取数据或丢失数据。
- *同步*队列不允许溢出，并且只能有一个读者。

两种队列类型都不允许下溢（从空队列读取将失败）并且只能有一个写入器。

### 不同步

非同步队列只有一个写入者，但可以有任意数量的读取者。队列有一个写位置；但是，每个阅读器都会跟踪自己独立的阅读位置。

只要队列容量不大于配置的队列容量（大于队列容量的写入立即失败），写入队列总是成功的（不检查溢出）。由于每个读者可能有不同的读取位置，而不是等待每个读者读取每条数据，只要新的写入需要空间，就允许数据从队列中掉下来。

读者负责在数据离开队列末尾之前检索数据。尝试读取比可用数据更多的数据的读取要么立即失败（如果非阻塞），要么等待足够的数据可用（如果阻塞）。尝试读取比队列容量更多的数据的读取总是立即失败。

如果一个reader跟不上writer，以至于那个reader已经写入但还未读取的数据量大于队列容量，下一次读取不返回数据；相反，它将读取器的读取位置重置为等于最新的写入位置，然后返回失败。如果在溢出之后、下一次读取之前检查可供读取的数据，则显示可供读取的数据多于队列容量，则表明发生了溢出。 （如果队列在检查可用数据和尝试读取该数据之间溢出，则溢出的唯一指示是读取失败。）

非同步队列的读者可能不想重置队列的读写指针。因此，当从描述符创建队列时，读者应该为 `resetPointers` 参数使用 `false` 参数。

### 同步的

同步队列有一个写入器和一个读取器，具有一个写入位置和一个读取位置。不可能写入超过队列空间的数据或读取超过队列当前持有的数据。根据调用的是阻塞还是非阻塞写入或读取函数，尝试超出可用空间或数据时，要么立即返回失败，要么阻塞，直到可以完成所需的操作。尝试读取或写入超过队列容量的数据总是会立即失败。

## 设置 FMQ

消息队列需要多个`MessageQueue`对象：一个要写入，一个或多个要从中读取。没有明确配置哪个对象用于写入或读取；由用户来确保没有对象同时用于读取和写入，最多只有一个写入者，并且对于同步队列，最多有一个读取者。

### 创建第一个 MessageQueue 对象

通过一次调用创建并配置消息队列：

```cpp
#include <fmq/MessageQueue.h>
using android::hardware::kSynchronizedReadWrite;
using android::hardware::kUnsynchronizedWrite;
using android::hardware::MQDescriptorSync;
using android::hardware::MQDescriptorUnsync;
using android::hardware::MessageQueue;
....
// For a synchronized non-blocking FMQ
mFmqSynchronized =
  new (std::nothrow) MessageQueue<uint16_t, kSynchronizedReadWrite>
      (kNumElementsInQueue);
// For an unsynchronized FMQ that supports blocking
mFmqUnsynchronizedBlocking =
  new (std::nothrow) MessageQueue<uint16_t, kUnsynchronizedWrite>
      (kNumElementsInQueue, true /* enable blocking operations */);
```

- `MessageQueue<T, flavor>(numElements)`初始化器创建并初始化一个支持消息队列功能的对象。
- `MessageQueue<T, flavor>(numElements, configureEventFlagWord)`初始化器创建并初始化一个对象，该对象支持带阻塞的消息队列功能。
- `flavor`可以是同步队列的`kSynchronizedReadWrite`或非同步队列的`kUnsynchronizedWrite` 。
- `uint16_t` （在此示例中）可以是任何[HIDL 定义的类型](https://source.android.com/docs/core/architecture/hidl-cpp/types)，不涉及嵌套缓冲区（无`string`或`vec`类型）、句柄或接口。
- `kNumElementsInQueue`指示队列的条目数大小；它确定将为队列分配的共享内存缓冲区的大小。

### 创建第二个 MessageQueue 对象

消息队列的第二端是使用从第一端获得的`MQDescriptor`对象创建的。 `MQDescriptor`对象通过 HIDL 或 AIDL RPC 调用发送到将保留消息队列第二端的进程。 `MQDescriptor`包含有关队列的信息，包括：

- 映射缓冲区和写入指针的信息。
- 映射读指针的信息（如果队列是同步的）。
- 映射事件标志字的信息（如果队列阻塞）。
- 对象类型 ( `<T, flavor>` )，包括[HIDL 定义的队列元素类型](https://source.android.com/docs/core/architecture/hidl-cpp/types)和队列风格（同步或非同步）。

`MQDescriptor`对象可用于构造`MessageQueue`对象：

```cpp
MessageQueue<T, flavor>::MessageQueue(const MQDescriptor<T, flavor>& Desc, bool resetPointers)
```

`resetPointers`参数表示创建这个`MessageQueue`对象时是否将读写位置重置为0。在非同步队列中，读取位置（非同步队列中每个`MessageQueue`对象的本地）在创建期间始终设置为 0。通常， `MQDescriptor`在创建第一个消息队列对象期间被初始化。为了对共享内存进行额外控制，您可以手动设置`MQDescriptor` （ `MQDescriptor`在[`system/libhidl/base/include/hidl/MQDescriptor.h`](https://android.googlesource.com/platform/system/libhidl/+/master/base/include/hidl/MQDescriptor.h)中定义），然后创建每个`MessageQueue`对象，如本节所述。

### 阻塞队列和事件标志

默认情况下，队列不支持阻塞读/写。有两种阻塞读/写调用：

- 短格式

  ，具有三个参数（数据指针、项目数、超时）。支持阻塞单个队列上的单个读/写操作。使用这种形式时，队列将在内部处理事件标志和位掩码，并且第

  一个消息队列对象

  必须使用第二个参数

  ```
  true
  ```

  进行初始化。例如：

  // For an unsynchronized FMQ that supports blocking mFmqUnsynchronizedBlocking =  new (std::nothrow) MessageQueue<uint16_t, kUnsynchronizedWrite>      (kNumElementsInQueue, true /* enable blocking operations */);

  ```
  
  ```

- *Long form*, with six parameters (includes event flag and bitmasks). Supports using a shared `EventFlag` object between multiple queues and allows specifying the notification bit masks to be used. In this case, the event flag and bitmasks must be supplied to each read and write call.

For the long form, the `EventFlag` can be supplied explicitly in each `readBlocking()` and `writeBlocking()` call. One of the queues may be initialized with an internal event flag, which must then be extracted from that queue's `MessageQueue` objects using `getEventFlagWord()` and used to create `EventFlag` objects in each process for use with other FMQs. Alternatively, the `EventFlag` objects can be initialized with any suitable shared memory.

In general, each queue should use only one of non-blocking, short-form blocking, or long-form blocking. It is not an error to mix them, but careful programming is required to get the desired result.

## Using the MessageQueue

The public API of the `MessageQueue` object is:

```cpp
size_t availableToWrite()  // Space available (number of elements).
size_t availableToRead()  // Number of elements available.
size_t getQuantumSize()  // Size of type T in bytes.
size_t getQuantumCount() // Number of items of type T that fit in the FMQ.
bool isValid() // Whether the FMQ is configured correctly.
const MQDescriptor<T, flavor>* getDesc()  // Return info to send to other process.

bool write(const T* data)  // Write one T to FMQ; true if successful.
bool write(const T* data, size_t count) // Write count T's; no partial writes.

bool read(T* data);  // read one T from FMQ; true if successful.
bool read(T* data, size_t count);  // Read count T's; no partial reads.

bool writeBlocking(const T* data, size_t count, int64_t timeOutNanos = 0);
bool readBlocking(T* data, size_t count, int64_t timeOutNanos = 0);

// Allows multiple queues to share a single event flag word
std::atomic<uint32_t>* getEventFlagWord();

bool writeBlocking(const T* data, size_t count, uint32_t readNotification,
uint32_t writeNotification, int64_t timeOutNanos = 0,
android::hardware::EventFlag* evFlag = nullptr); // Blocking write operation for count Ts.

bool readBlocking(T* data, size_t count, uint32_t readNotification,
uint32_t writeNotification, int64_t timeOutNanos = 0,
android::hardware::EventFlag* evFlag = nullptr) // Blocking read operation for count Ts;

//APIs to allow zero copy read/write operations
bool beginWrite(size_t nMessages, MemTransaction* memTx) const;
bool commitWrite(size_t nMessages);
bool beginRead(size_t nMessages, MemTransaction* memTx) const;
bool commitRead(size_t nMessages);
```

`availableToWrite()`和`availableToRead()`可用于确定在单个操作中可以传输多少数据。在非同步队列中：

- `availableToWrite()`总是返回队列的容量。
- 每个阅读器都有自己的阅读位置，并对`availableToRead()`进行自己的计算。
- 从慢读者的角度来看，允许队列溢出；这可能会导致`availableToRead()`返回一个大于队列大小的值。溢出后的第一次读取将失败，并导致该读取器的读取位置设置为等于当前写入指针，无论是否通过`availableToRead()`报告了溢出。

如果所有请求的数据都可以（并且已经）传输到队列/从队列传输，则`read()`和`write()`方法返回`true` 。这些方法不会阻塞；他们要么成功（并返回`true` ），要么立即返回失败（ `false` ）。

`readBlocking()`和`writeBlocking()`方法一直等到请求的操作可以完成，或者直到它们超时（ `timeOutNanos`值为 0 意味着永远不会超时）。

阻塞操作是使用事件标志字实现的。默认情况下，每个队列创建并使用自己的标志字来支持`readBlocking()`和`writeBlocking()`的缩写形式。多个队列可以共享一个字，因此进程可以等待对任何队列的写入或读取。可以通过调用`getEventFlagWord()`获得指向队列事件标志字的指针，并且该指针（或指向合适的共享内存位置的任何指针）可用于创建`EventFlag`对象以传递到长格式的`readBlocking()`和`writeBlocking()`用于不同的队列。 `readNotification`和`writeNotification`参数告诉应该使用事件标志中的哪些位来指示对该队列的读取和写入。 `readNotification`和`writeNotification`是 32 位位掩码。

`readBlocking()`等待`writeNotification`位；如果该参数为 0，则调用总是失败。如果`readNotification`值为 0，则调用不会失败，但成功读取不会设置任何通知位。在同步队列中，这意味着相应的`writeBlocking()`调用永远不会唤醒，除非该位在别处设置。在非同步队列中， `writeBlocking()`不会等待（应该还是用来设置写通知位），读不设置任何通知位是合适的。同样，如果`readNotification`为 0，则`writeblocking()`将失败，并且成功的写入会设置指定的`writeNotification`位。

要同时等待多个队列，请使用`EventFlag`对象的`wait()`方法来等待通知的位掩码。 `wait()`方法返回一个状态字，其中包含导致唤醒设置的位。然后使用此信息来验证相应的队列是否有足够的空间或数据来进行所需的写/读操作，并执行非阻塞的`write()` / `read()` 。要获得操作后通知，请再次调用`EventFlag`的`wake()`方法。有关`EventFlag`抽象的定义，请参阅[`system/libfmq/include/fmq/EventFlag.h`](https://android.googlesource.com/platform/system/libfmq/+/master/include/fmq/EventFlag.h) 。

## 零拷贝操作

`read` / `write` / `readBlocking` / `writeBlocking()` API 将指向输入/输出缓冲区的指针作为参数，并在内部使用`memcpy()`调用在相同缓冲区和 FMQ 环形缓冲区之间复制数据。为了提高性能，Android 8.0 及更高版本包含一组 API，可提供对环形缓冲区的直接指针访问，从而无需使用`memcpy`调用。

使用以下公共 API 进行零复制 FMQ 操作：

```cpp
bool beginWrite(size_t nMessages, MemTransaction* memTx) const;
bool commitWrite(size_t nMessages);

bool beginRead(size_t nMessages, MemTransaction* memTx) const;
bool commitRead(size_t nMessages);
```

- `beginWrite`方法提供指向 FMQ 环形缓冲区的基址指针。写入数据后，使用`commitWrite()`提交它。 `beginRead` / `commitRead`方法的作用相同。
- `beginRead` / `Write`方法将要读/写的消息数作为输入，并返回一个布尔值，指示是否可以读/写。如果可以读取或写入，则`memTx`结构将填充有可用于直接指针访问环形缓冲区共享内存的基指针。
- `MemRegion`结构包含有关内存块的详细信息，包括基址指针（内存块的基地址）和以`T`表示的长度（以 HIDL 定义的消息队列类型表示的内存块长度）。
- `MemTransaction`结构包含两个`MemRegion`结构， `first`和`second` ，因为读取或写入环形缓冲区可能需要环绕到队列的开头。这意味着需要两个基指针来将数据读/写到 FMQ 环形缓冲区中。

从`MemRegion`结构中获取基地址和长度：

```cpp
T* getAddress(); // gets the base address
size_t getLength(); // gets the length of the memory region in terms of T
size_t getLengthInBytes(); // gets the length of the memory region in bytes
```

要获取对`MemTransaction`对象中第一个和第二个`MemRegion`的引用：

```cpp
const MemRegion& getFirstRegion(); // get a reference to the first MemRegion
const MemRegion& getSecondRegion(); // get a reference to the second MemRegion
```

使用零复制 API 写入 FMQ 的示例：

```cpp
MessageQueueSync::MemTransaction tx;
if (mQueue->beginRead(dataLen, &tx)) {
    auto first = tx.getFirstRegion();
    auto second = tx.getSecondRegion();

    foo(first.getAddress(), first.getLength()); // method that performs the data write
    foo(second.getAddress(), second.getLength()); // method that performs the data write

    if(commitWrite(dataLen) == false) {
       // report error
    }
} else {
   // report error
}
```

以下辅助方法也是`MemTransaction`的一部分：

- `T* getSlot(size_t idx);`
  返回指向作为此`MemTransaction`对象一部分的`MemRegions`中的插槽`idx`的指针。如果`MemTransaction`对象表示要读/写 N 项类型 T 的内存区域，则`idx`的有效范围在 0 到 N-1 之间。
- `bool copyTo(const T* data, size_t startIdx, size_t nMessages = 1);`
  将类型 T 的`nMessages`项写入对象描述的内存区域，从索引`startIdx`开始。此方法使用`memcpy()`并且不打算用于零复制操作。如果`MemTransaction`对象表示要读/写 N 个类型 T 的内存，则`idx`的有效范围在 0 到 N-1 之间。
- `bool copyFrom(T* data, size_t startIdx, size_t nMessages = 1);`
  从`startIdx`开始的对象所描述的内存区域中读取 T 类型的`nMessages`项的辅助方法。此方法使用`memcpy()`并且不打算用于零复制操作。

## 通过 HIDL 发送队列

在创作方面：

1. 如上所述创建消息队列对象。
2. 使用`isValid()`验证对象是否有效。
3. 如果您将通过将`EventFlag`传递到长格式的`readBlocking()` / `writeBlocking()`来等待多个队列，您可以从初始化为创建标志的`MessageQueue`对象中提取事件标志指针（使用`getEventFlagWord()` ），并使用该标志创建必要的`EventFlag`对象。
4. 使用`MessageQueue` `getDesc()`方法获取描述符对象。
5. 在`.hal`文件中，为该方法提供一个`fmq_sync`类型的参数或者`fmq_unsync`其中`T`是合适的 HIDL 定义类型。使用它来将`getDesc()`返回的对象发送到接收进程。

在接收端：

1. 使用描述符对象创建`MessageQueue`对象。请务必使用相同的队列风格和数据类型，否则模板将无法编译。
2. 如果提取了事件标志，则在接收进程中从相应的`MessageQueue`对象中提取标志。
3. 使用`MessageQueue`对象传输数据。