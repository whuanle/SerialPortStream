# Serial Port Stream

SerialPortStream is an independent implementation of
`System.IO.Ports.SerialPort` and `SerialStream` for better reliability and
maintainability, and now for portability to Mono on Linux systems.

The `SerialPortStream` is a ground up implementation of a Stream that buffers
data to and from a serial port. It uses low level Win32API (on Windows) or Posix
(on Linux) for managing events and asynchronous I/O, using a programming model
as in the MSDN
[PipeServer](http://msdn.microsoft.com/en-us/library/windows/desktop/aa365603.aspx)
example.

These notes are for version 2.x, which is a new design based on version 1.x that
enhances portability and fixes bugs. See the end of these notes for differences.

* 1.0 Why another Serial Port implementation
* 2.0 Goals
  * 2.1 Issues with MS Serial Port
* 3.0 System Requirements
  * 3.1 Tested
  * 3.2 Compatibility
    * 3.2.1 .NET Frameworks (Windows)
    * 3.2.2 Mono Framework (Linux Only)
    * 3.2.3 Microsoft Compact Framework (not supported)
* 4.0 Installation
  * 4.1 Windows
  * 4.2 Linux
* 5.0 Extra Features
  * 5.1 Reading and Writing - Buffering
* 6.0 Known Issues
  * 6.1 Windows
    * 6.1.1 Driver Specific Issues on Windows
      * 6.1.1.1 Flow Control
  * 6.2 Linux
    * 6.2.1 Mono on non-Windows Platforms
    * 6.2.2 Driver Specific Issues on Linux
      * 6.2.2.1 Parity Errors
      * 6.2.2.2 Garbage Data on Open
      * 6.2.2.3 Monitoring Pins and Timing Resolution
      * 6.2.2.4 Close Times with Flow Control

## 1.0 Why another Serial Port implementation

Microsoft and Mono already provides a reasonable implementation for accessing
the serial port. Unfortunately, documentation is sparse. When one tries to find
information about how to program the serial port one comes across instead many
blogs and forums describing the issues that they've observed.

Through the implementation of `SerialPortStream`, I've used
[ILSpy](http://ilspy.net/) to reverse engineer how the Microsoft implementation
works, discovering many other subtle, but noteworthy implementation issues.

## 2.0 Goals

This project tries to achieve the following:

* An implementation similar to the MS implementation of `SerialPort`. It's not
  meant to be 100% compatible, but instead provide similar functionality
* Abstract the driver implementation and provide for a more reliable transport,
  by making writing serial data completely buffered. With the MS implementation,
  one can write data, but subsequently needs to check if all data is written or
  not. If it isn't written, then it needs to be retried. The `SerialPortStream`
  makes this easier.
* Provide for reliable and consistent behaviour. See the next section.

### 2.1 Issues with MS Serial Port

The `SerialPortStream` tries to solve the following issues observed:

* Zach Saw describes issues regarding behaviour of the `fAbortError` flag in the
  Serial `DCB`. The `SerialPortStream` defines this flag.
* Closing a serial port, then reopening it generally causes problems. The
  `SerialPortStream` shouldn't have this issue. Note, there are some cases
  observed where the Operating System hangs, and this can't be avoided.
* The `ReadTo()` implementation can subtly change the byte stream buffer, when
  one switches from characters to bytes. This problem occurs because the MS
  implementation actually converts the characters back to bytes into its buffer.
  So if you have UTF8, decoded some invalid characters and then have a timeout,
  this results in the invalid characters being converted back to bytes,
  resulting in "lost" data. I take some care when decoding bytes to characters
  to ensure a seamless and accurate transition between bytes and characters.
* `Write()` gives the data to the serial port. If the operation is asynchronous,
  the callback results in the number of bytes that were actually transferred to
  the driver. You need to check yourself if this is valid or not. In the
  synchronous case, the data is simply thrown away. The `SerialPortStream`
  method simply copies data to a local buffer and uses asynchronous writes in a
  different thread. It works in the background to send out the data you
  provided. If the data can't be sent, then you get a `TimeoutException` without
  any data being buffered at all. So you can implement reliable protocols and
  your code is simpler.
* Disposing or Closing the serial port during a blocking write operation will
  not abort the write operation. This implementation will abort with an
  `System.IO.IOException` type.

## 3.0 System Requirements

### 3.1 Tested

Software has been tested and developed using:

* .NET Standard 1.5 on Windows 10 Pro x64
* .NET 4.8 on Windows 10 Pro x64
* .NET 4.5 on Windows 8 Pro x64 and Windows 8.1 Pro x64.
* .NET 4.5 on Windows 7 x86 and x64.
* Mono 6.x from Xamarin on Ubuntu 18.04 (64-bit)

I use this software for automation in another system (Windows) that runs for
multiple days and it appears stable.

See later in these notes for known issues and changes.

### 3.2 Compatibility

#### 3.2.1 .NET Frameworks (Windows)

The software is written originally for .NET 4.0 and should work on those
platforms. It is extended for .NET 4.5 features. A version targets .NET Core
with API level .NET Stanrdard 1.5, so should work on .NET Core 2.1, 3.1 and .NET
5.0 and later.

Windows XP SP3 and later should work.

#### 3.2.2 Mono Framework (Linux Only)

TBD: Since the migration to the .NET SDK project format, this is not tested.

Mono on Linux is supported. Embedded platforms are not supported. You are
welcome to make a patch with appropriate test cases.

You should use the latest version of Mono. Version 3.2.8 has significant bugs
and will not work (Ubuntu 14.04 ships with this). Use the latest version of Mono
that comes direct from Xamarin instead of your distribution where possible.

For instructions on how to install the latest Mono for your system, refer to
[Install Mono On
Linux](http://www.mono-project.com/docs/getting-started/install/linux/).

#### 3.2.3 Microsoft Compact Framework (not supported)

SerialPortStream is not designed for the Compact Framework.

## 4.0 Installation

### 4.1 Windows

On Windows, just reference the assembly in your project installing the NuGet
version.

### 4.2 Linux

You first need to compile the support library `libnserial.so` for your platform.
To do that, you'll need a compiler (e.g. GCC 4.8 or later) and `cmake`. The
binaries for Linux are not part of the distribution, as it's operating system
specific.

After cloning the repository, execute the following:

```sh
git clone https://github.com/jcurl/serialportstream.git
cd serialportstream/dll/serialunix
./build.sh
```

Binaries are built and put in the `bin` folder from where you ran the build
script. You can add a reference to `LD_LIBRARY_PATH` to the library:

```sh
export LD_LIBRARY_PATH=`pwd`/bin/usr/local/lib:$LD_LIBRARY_PATH
```

and then run your Mono program from there.

Or you can build and install in your system:

```sh
cd serialportstream/dll/serialunix
mkdir mybuild
cd mybuild
cmake .. && make
sudo make install
```

## 5.0 Extra Features

The following features are in addition to the `System.IO.Ports.SerialPort`
implementation:

* You can obtain the RingIndicator pin status.
* The `Read()` and `Write()` buffers are completely independent of the low level
  Windows driver.

### 5.1 Reading and Writing - Buffering

Why is it interesting to perform buffering? A driver might be configured to be
4096 or 8192 bytes (which is quite typical). Testing with the PL2303 chipset,
one can't write more than about 12KB with a single write operation.

A Write buffer may be 128KB, which one writes to. The thread in the background
will write the data and issue as many write calls as is necessary to get the job
done. A Read buffer may be 5MB. The background thread will read from the serial
port when ever data arrives and buffers into the 5MB.

So long as the I/O thread in .NET an execute every 100-200ms, it can continue to
read data from the driver. Your own application doesn't need to keep to such
difficult time constraints. Such issues typically arise in Automation type
environments where a computer has many different peripherals. So long as the
process doesn't block, your main application might sleep for 10 seconds and
you've still lost no data. The MS implementation wouldn't be so simple, you have
to make sure that you perform frequent read operations else the driver itself
might overflow (resulting in lost data).

## 6.0 Known Issues

### 6.1 Windows

The following issues are known:

* This is not an issue, but when using the `Com0Com` for running unit tests,
  some specific test cases for Parity will fail. That is because Com0Com doesn't
  emulate data at a bit level.
* .NET 4.0 to the currently tested .NET 4.6 has a minor bug in
  System.Text.Decoder that in a special circumstance it will consume too many
  bytes. The `PeekChar()` method is therefore a slower implementation than what
  it could be. Please refer to the Xamarin bug
  [40002](https://bugzilla.xamarin.com/show_bug.cgi?id=40002). Found against
  Mono 4.2.3.4 and later tested to be present since .NET 4.0 on Windows XP also.

#### 6.1.1 Driver Specific Issues on Windows

##### 6.1.1.1 Flow Control

Using the FTDI chipset on Windows 10 x64 (FTDI 2.12.16.0 dated 09/Mar/2016) flow
control (RTS/CTS) doesn't work as expected. For writing small amounts of data
(1024 bytes) with CTS off, the FTDI driver will still send data. See the test
case ClosedWhenFlushBlocked, change the buffer from 8192 bytes to 1024 and the
test case now fails. This problem is not observable with com0com 3.0. You can
see the effect in logs, there is a TX-EMPTY event that occurs, which should
never be there if no data is ever sent.

### 6.2 Linux

SerialPortStream was tested on Ubuntu 14.04 and Ubuntu 16.04. Feedback welcome
for other distributions!

The main functionality on Linux is provided by a support C library. The issues
are observed:

* Custom baud rates are not supported. To know what baudrates are supported on
  your system, look at the file `config.h` after building.
* DSR and DTR handshaking is not supported. You can still set and clear the pins
  though.

Patches are welcome to implement these features!

#### 6.2.1 Mono on non-Windows Platforms

Ubuntu 14.04 ships with Mono 3.2.8. This is known to not work.

* [[Mono-Dev] Mono 3.2.8 incompatibility with .NET 4.0 on Windows
  7-10](http://lists.ximian.com/pipermail/mono-devel-list/2015-December/043423.html).
  The System.Text implementation for converting bytes to UTF8 don't work. If you
  don't use the character based methods, it may work. But the software has not
  been tested against this framework.
* The DataReceived event doesn't fire for the EOF character (0x1A). On WIndows
  it does, as this is managed by the driver itself.
* The test case for opening two serial ports simultaneously on Mono fails,
  meaning that it's possible to open the same device twice, which on Windows
  raises an UnauthorizedAccessException().
* ListPorts is not implemented on Mono and uses the SerialPort implementation.
* Mono 4.2.3.4 (tested) has a minor bug in System.Text.Decoder as in the .NET
  references, that in a special circumstance it will consume too many bytes. The
  PeekChar() method is slower when this bug is detected. Please refer to the
  Xamarin bug [40002](https://bugzilla.xamarin.com/show_bug.cgi?id=40002). Found
  against Mono 4.2.3.4 and later tested to be present since .NET 4.0 on Windows
  XP also.

#### 6.2.2 Driver Specific Issues on Linux

Tests have been done using FTDO, PL2303H, PL2303RA and 16550A (some still do
exist!).  The following has been observed:

##### 6.2.2.1 Parity Errors

Some chipsets do not report properly parity errors. The 16550A chipset works as
expected. Issues observed with FTDI, PL2303H, PL2303RA. In particular, on a
parity error, more bytes are reported as having parity errors than there are in
the stream. Tested using loopback devices with 'comptest'.

```sh
$ ./nserialcomptest /dev/ttyUSB0 /dev/ttyUSB1`
  [ RUN      ] SerialParityTest.Parity7E1ReceiveError
/home/jcurl/Programming/serialportstream/dll/serialunix/libnserial/comptest/SerialParityTest.cpp:221: Failure
Value of: comparison
  Actual: false
Expected: true
Unexpected byte received with Even Parity
[  FAILED  ] SerialParityTest.Parity7E1ReceiveError (585 ms)
[ RUN      ] SerialParityTest.Parity7O1ReceiveError
/home/jcurl/Programming/serialportstream/dll/serialunix/libnserial/comptest/SerialParityTest.cpp:373: Failure
Value of: comparison
  Actual: false
Expected: true
Unexpected byte received with Even Parity
[  FAILED  ] SerialParityTest.Parity7O1ReceiveError (584 ms)
[ RUN      ] SerialParityTest.Parity7O1ReceiveErrorWithReplace
/home/jcurl/Programming/serialportstream/dll/serialunix/libnserial/comptest/SerialParityTest.cpp:427: Failure
Value of: comparison
  Actual: false
Expected: true
Unexpected byte received with Even Parity
[  FAILED  ] SerialParityTest.Parity7O1ReceiveErrorWithReplace (572 ms)
```

##### 6.2.2.2 Garbage Data on Open

On Linux Kernel with Ubuntu 14.04 and Ubuntu 16.04, we observe that some USB-SER
drivers provide extra data depending on what a previous process was doing. It
shows itself as garbage zero's appearing at the beginning of a stream when
reading, and may be visible in your application also. There's a test case
`comptest/kernelbug` that shows this behaviour on a Lenevo T61p. Affected is
PL2303H and FTDI chipsets. Chipsets that don't show this behaviour are 16550A
and PL2303RA chipsets. Invocate the test program twice and you'll see the error.
This is reported to
[Ubuntu](https://bugs.launchpad.net/ubuntu/+source/linux/+bug/1542862)

```sh
$ kernelbug /dev/ttyUSB0 /dev/ttyUSB1
Offset: 4
Flushing...
Writing Complete...
Reading complete...
Comparison MATCH                    <---- PASS
Flushing...
Reading complete...
Complete...

$ kernelbug /dev/ttyUSB0 /dev/ttyUSB1
Offset: 108
Flushing...
Flush 2 bytes
Writing Complete...
Reading complete...
ERROR: Comparison mismatch          <---- ERROR
Flushing...
Flush 510 bytes
Reading complete...
Complete...
```

##### 6.2.2.3 Monitoring Pins and Timing Resolution

Monitoring of pins CTS, DSR, RI and DCD is not 100% reliable for some chipsets
and workarounds are in place. In particular, the chips PL2303H, PL2303RA do not
support the ioctl(TIOCGICOUNT), so on a pin toggle, we cannot reliably detect if
they have changed if the pulse is too short. For 16550A and FTDI chips, this
ioctl() does work and so we can always detect a change. To check if your driver
supports the TIOCGICOUNT ioctl() call, run the small test program
`comptest/icount`.

```sh
$ ./icount /dev/ttyS0
Your driver supports TIOCGICOUNT
ocounter.cts=0
ocounter.dsr=0
ocounter.rng=3
ocounter.dcd=0
```

or in the case it's not supported:

```sh
$ ./icount /dev/ttyUSB0
Your driver doesn't support TIOCGICOUNT
  Error: 25 (Inappropriate ioctl for device)
```

##### 6.2.2.4 Close Times with Flow Control

Some times closing the serial port may take a long time (observed from 5s to
21s) if it is write blocked due to hardware flow control. In particular, the
C-library function `serial_close()` appears to take an excessive time when
calling `close(handle->fd)` on Ubuntu 16.04. This issue appears related to the
Linux driver and not the MONO framework.

The .NET Test Cases that show this behaviour are (blocking on write):

* `ClosedWhenBlocked`
* `CloseWhenFlushBlocked`
* `DisposeWhenBlocked`
* `DisposeWhenFlushBlocked`

This issue is not reproducible with the 16550A UART when it is write blocked. In
this case, the times for closing are usually not more than 20ms.
