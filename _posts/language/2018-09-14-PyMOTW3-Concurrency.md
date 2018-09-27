---
title: PyMOTW-3 --- Concurrency with Processes, Threads, and Coroutines
categories: language
tags: PyMOTW-3
---

Python包含“使用进程和线程”管理并发操作的复杂工具。通过使用这些模块同时运行部分作业的技术，甚至可以使许多相对简单的程序运行得更快。

subprocess提供用于创建辅助进程和与其通信的API。它特别适合运行生成或使用文本的程序，因为API支持通过新进程的标准输入和输出通道来回传递数据。

信号模块公开Unix信号机制，用于将事件发送到其他进程。信号是异步处理的，通常是通过中断当信号到达时程序正在做的事情来实现的。信号作为一个粗糙的消息系统是有用的，但是其他进程间通信技术更可靠并且可以传递更复杂的消息。

threading包括一个高级的，面向对象的API，用于处理Python的并发。Thread对象在同一进程中并发运行并共享内存。对于受`I/O`限制比CPU限制更多的任务，使用线程是一种简单的扩展方法。

multiprocessing模块镜像threading，除了它提供一个Process而不是Thread类。每个Process都是一个真正系统进程，不共享内存，但multiprocessing提供了共享数据和在它们之间传递消息的功能，因此在许多情况下从线程转换到进程就像更改一些import语句一样简单。

asyncio使用基于类的协议系统或协程提供并发和异步`I/O`管理的框架。asyncio取代了旧的asyncore和asynchat模块，这些模块仍然可用但不建议使用。

concurrent.futures提供了线程和基于进程的执行程序的实现，用于管理用于运行并发任务的资源池。

## subprocess — Spawning Additional Processes

目的：启动其他进程并与其进行通信。  
subprocess模块支持三个用于处理进程的API。在Python3.5中添加的run（）函数是一个用于运行进程并可选地收集其输出的高级API。函数call（），`check_call（）`和`check_output（）`是以前的高级API，从Python2继承而来。它们仍然受支持并在现有程序中广泛使用。Popen类是一个低级API，用于构建其他API，对更复杂的进程交互很有用。 Popen的构造函数接受参数来设置新进程，以便父进程可以通过管道与它通信。它提供了其替代的其他模块和功能的所有功能，而且更多。API对于所有用途都是一致的，并且需要许多额外的开销步骤（例如关闭额外的文件描述符并确保管道关闭）是“内置的”而不是由应用程序代码单独处理。

subprocess模块旨在替换os.system（），os.spawnv（），os和popen2模块中popen（）的变体以及commands（）等函数。为了便于将subprocess与其他模块进行比较，本节中的许多示例都重新创建了用于os和popen2的示例。

>注意  
在Unix和Windows上工作的API大致相同，但由于操作系统中的进程模型不同，底层实现也不同。此处显示的所有示例均在Mac OS X上进行了测试。非Unix操作系统上的行为可能会有所不同。

### Running External Command

要以与os.system（）相同的方式运行外部命令而不与其交互，请使用run（）函数。

```python
#subprocess_os_system.py
import subprocess

completed = subprocess.run(['ls', '-1'])
print('returncode:', completed.returncode)
```
命令行参数作为字符串列表传递，这避免了“转义引号或shell可能解释的其他特殊字符”的需要。 run（）返回一个CompletedProcess实例，其中包含有关进程的信息，如退出码和输出。

```
$ python3 subprocess_os_system.py

index.rst
interaction.py
repeater.py
subprocess_os_system.py
...
returncode: 0
```
将shell参数设置为true值会导致subprocess生成一个中间shell进程运行该命令。 默认是直接运行命令。

```python
#subprocess_shell_variables.py
import subprocess

completed = subprocess.run('echo $HOME', shell=True)
print('returncode:', completed.returncode)
```
使用中间shell意味着在命令运行之前处理命令字符串中的变量，glob模式和其他特殊shell特性。

```
$ python3 subprocess_shell_variables.py

/Users/dhellmann
returncode: 0
```
>注意  
使用run（）而不传递 check=True 等效于使用call（），它只返回进程的退出码。

#### Error Handling

CompletedProcess的returncode属性是程序的退出码。调用者负责解释它以检测错误。如果run（）的check参数为True，则退出码被检查，如果它指示发生错误，则引发CalledProcessError异常。

```python
#subprocess_run_check.py
import subprocess

try:
    subprocess.run(['false'], check=True)
except subprocess.CalledProcessError as err:
    print('ERROR:', err)
```
false命令总是以非零状态代码退出，run（）将其解释为错误。

```
$ python3 subprocess_run_check.py

ERROR: Command '['false']' returned non-zero exit status 1
```
>注意  
将 check=True 传递给run（）使其等效于使用check_call（）。

#### Capturing Output

run（）启动的进程的标准输入和输出通道绑定到父进程的输入和输出。这意味着调用程序无法捕获命令的输出。传递PIPE给stdout和stderr参数以捕获输出以供稍后处理。

```python
#subprocess_run_output.py
import subprocess

completed = subprocess.run(
    ['ls', '-1'],
    stdout=subprocess.PIPE,
)
print('returncode:', completed.returncode)
print('Have {} bytes in stdout:\n{}'.format(
    len(completed.stdout),
    completed.stdout.decode('utf-8'))
)
```
ls -1命令成功运行，因此打印到标准输出的文本被捕获并返回。

```
$ python3 subprocess_run_output.py

returncode: 0
Have 522 bytes in stdout:
index.rst
interaction.py
repeater.py
signal_child.py
signal_parent.py
subprocess_check_output_error_trap_output.py
subprocess_os_system.py
subprocess_pipes.py
subprocess_popen2.py
subprocess_popen3.py
subprocess_popen4.py
subprocess_popen_read.py
subprocess_popen_write.py
subprocess_run_check.py
subprocess_run_output.py
subprocess_run_output_error.py
subprocess_run_output_error_suppress.py
subprocess_run_output_error_trap.py
subprocess_shell_variables.py
subprocess_signal_parent_shell.py
subprocess_signal_setpgrp.py
```
>注意
传递 check=True并将stdout设置为PIPE等同于使用check_output（）。

下一个示例在子shell中运行一系列命令。在命令以一个错误码退出之前，消息将发送到标准输出和标准错误。

```python
#subprocess_run_output_error.py
import subprocess

try:
    completed = subprocess.run(
        'echo to stdout; echo to stderr 1>&2; exit 1',
        check=True,
        shell=True,
        stdout=subprocess.PIPE,
    )
except subprocess.CalledProcessError as err:
    print('ERROR:', err)
else:
    print('returncode:', completed.returncode)
    print('Have {} bytes in stdout: {!r}'.format(
        len(completed.stdout),
        completed.stdout.decode('utf-8'))
    )
```
标准错误消息将打印到控制台，但到标准输出的消息被隐藏。

```
$ python3 subprocess_run_output_error.py

to stderr
ERROR: Command 'echo to stdout; echo to stderr 1>&2; exit 1'
returned non-zero exit status 1
```
要防止通过run（）运行的命令的错误消息被写入控制台，请将stderr参数设置为常量PIPE。

```python
#subprocess_run_output_error_trap.py
import subprocess

try:
    completed = subprocess.run(
        'echo to stdout; echo to stderr 1>&2; exit 1',
        shell=True,
        stdout=subprocess.PIPE,
        stderr=subprocess.PIPE,
    ) 
except subprocess.CalledProcessError as err:
    print('ERROR:', err)
else:
    print('returncode:', completed.returncode)
    print('Have {} bytes in stdout: {!r}'.format(
        len(completed.stdout),
        completed.stdout.decode('utf-8'))
    )
    print('Have {} bytes in stderr: {!r}'.format(
        len(completed.stderr),
        completed.stderr.decode('utf-8'))
    )
```
此示例未设置 check=True，因此捕获并打印命令的输出。

```
$ python3 subprocess_run_output_error_trap.py

returncode: 1
Have 10 bytes in stdout: 'to stdout\n'
Have 10 bytes in stderr: 'to stderr\n'
```
要在使用check_output（）时捕获错误消息，请将stderr设置为STDOUT，并且此消息将与命令的其余输出合并。

```python
#subprocess_check_output_error_trap_output.py
import subprocess

try:
    output = subprocess.check_output(
        'echo to stdout; echo to stderr 1>&2',
        shell=True,
        stderr=subprocess.STDOUT,
    )
except subprocess.CalledProcessError as err:
    print('ERROR:', err)
else:
    print('Have {} bytes in output: {!r}'.format(
        len(output),
        output.decode('utf-8'))
    )
```
输出顺序可能会有所不同，具体取决于缓冲如何应用于标准输出流以及打印的数据量。

```
$ python3 subprocess_check_output_error_trap_output.py

Have 20 bytes in output: 'to stdout\nto stderr\n'
```

#### Suppressing Output

对于不应显示或捕获输出的情况，请使用DEVNULL来抑制输出流。 此示例禁止标准输出和错误流。

```python
#subprocess_run_output_error_suppress.py
import subprocess

try:
    completed = subprocess.run(
        'echo to stdout; echo to stderr 1>&2; exit 1',
        shell=True,
        stdout=subprocess.DEVNULL,
        stderr=subprocess.DEVNULL,
    )
except subprocess.CalledProcessError as err:
    print('ERROR:', err)
else:
    print('returncode:', completed.returncode)
    print('stdout is {!r}'.format(completed.stdout))
    print('stderr is {!r}'.format(completed.stderr))
```
名称DEVNULL来自Unix特殊设备文件`/dev/null`，它在打开读取和接收时以文件结尾响应，但在写入时忽略任何数量的输入。

```
$ python3 subprocess_run_output_error_suppress.py

returncode: 1
stdout is None
stderr is None
```

### Working with Pipes Directly

函数run()，call()，`check_call()`和`check_output()`是Popen类的包装器。直接使用Popen可以更好地控制命令的运行方式，以及如何处理输入和输出流。例如，通过为stdin，stdout和stderr传递不同的参数，可以模仿os.popen（）的变体。

#### One-way Communication With a Process

要运行进程并读取其所有输出，请将stdout值设置为PIPE并调用communicate()。

```python
#subprocess_popen_read.py
import subprocess

print('read:')
proc = subprocess.Popen(
    ['echo', '"to stdout"'],
    stdout=subprocess.PIPE,
)
stdout_value = proc.communicate()[0].decode('utf-8')
print('stdout:', repr(stdout_value))
```
这类似于popen（）的工作方式，除了读取由Popen实例内部管理。

```
$ python3 subprocess_popen_read.py

read:
stdout: '"to stdout"\n'
```
要设置管道以允许调用程序向其写入数据，请将stdin设置为PIPE。

```python
#subprocess_popen_write.py
import subprocess

print('write:')
proc = subprocess.Popen(
    ['cat', '-'],
    stdin=subprocess.PIPE,
)
proc.communicate('stdin: to stdin\n'.encode('utf-8'))
```
要将数据发送到进程的标准输入通道一次，请将数据传递给communication（）。 这与使用模式'w'的popen（）类似。

```
$ python3 -u subprocess_popen_write.py

write:
stdin: to stdin
```

#### Bi-directional Communication With a Process

要同时设置Popen实例进行读和写，请结合使用以前的技术。

```python
#subprocess_popen2.py
import subprocess

print('popen2:')

proc = subprocess.Popen(
    ['cat', '-'],
    stdin=subprocess.PIPE,
    stdout=subprocess.PIPE,
)
msg = 'through stdin to stdout'.encode('utf-8')
stdout_value = proc.communicate(msg)[0].decode('utf-8')
print('pass through:', repr(stdout_value))
```
这会将设置管道以模仿popen2（）。

```
$ python3 -u subprocess_popen2.py

popen2:
pass through: 'through stdin to stdout'
```

#### Capturing Error Output

与popen3() 一样，也可以监视stdout和stderr的两个流。

```python
#subprocess_popen3.py
import subprocess

print('popen3:')
proc = subprocess.Popen(
    'cat -; echo "to stderr" 1>&2',
    shell=True,
    stdin=subprocess.PIPE,
    stdout=subprocess.PIPE,
    stderr=subprocess.PIPE,
)
msg = 'through stdin to stdout'.encode('utf-8')
stdout_value, stderr_value = proc.communicate(msg)
print('pass through:', repr(stdout_value.decode('utf-8')))
print('stderr      :', repr(stderr_value.decode('utf-8')))
```
从stderr读取的工作与stdout相同。传递PIPE告诉Popen连接到此通道，并且communication（）在返回之前从中读取所有数据。

```
$ python3 -u subprocess_popen3.py

popen3:
pass through: 'through stdin to stdout'
stderr      : 'to stderr\n'
```

#### Combining Regular and Error Output

要将进程的错误输出定向到其标准输出通道，请使用STDOUT作为stderr而不是PIPE。

```python
#subprocess_popen4.py
import subprocess

print('popen4:')
proc = subprocess.Popen(
    'cat -; echo "to stderr" 1>&2',
    shell=True,
    stdin=subprocess.PIPE,
    stdout=subprocess.PIPE,
    stderr=subprocess.STDOUT,
)
msg = 'through stdin to stdout\n'.encode('utf-8')
stdout_value, stderr_value = proc.communicate(msg)
print('combined output:', repr(stdout_value.decode('utf-8')))
print('stderr value   :', repr(stderr_value))
```
以这种方式组合输出类似于popen4（）的工作方式。

```
$ python3 -u subprocess_popen4.py

popen4:
combined output: 'through stdin to stdout\nto stderr\n'
stderr value   : None
```

### Connecting Segments of a Pipe

通过创建单独的Popen实例并将它们的输入和输出链接在一起，可以将多个命令连接到管道中，类似于Unix shell的工作方式。一个Popen实例的stdout属性用作管道中下一个的stdin参数，而不是常量PIPE。从管道中的最后一个命令的stdout句柄读取输出。

```python
#subprocess_pipes.py
import subprocess

cat = subprocess.Popen(
    ['cat', 'index.rst'],
    stdout=subprocess.PIPE,
)

grep = subprocess.Popen(
    ['grep', '.. literalinclude::'],
    stdin=cat.stdout,
    stdout=subprocess.PIPE,
)

cut = subprocess.Popen(
    ['cut', '-f', '3', '-d:'],
    stdin=grep.stdout,
    stdout=subprocess.PIPE,
)

end_of_pipe = cut.stdout

print('Included files:')
for line in end_of_pipe:
    print(line.decode('utf-8').strip())
```
该示例重现命令行：

```
$ cat index.rst | grep ".. literalinclude" | cut -f 3 -d:
```
管道读取此部分的reStructuredText源文件，并查找包含其他文件的所有行，然后打印所包含文件的名称。

```
$ python3 -u subprocess_pipes.py

Included files:
subprocess_os_system.py
subprocess_shell_variables.py
subprocess_run_check.py
subprocess_run_output.py
subprocess_run_output_error.py
subprocess_run_output_error_trap.py
subprocess_check_output_error_trap_output.py
subprocess_run_output_error_suppress.py
subprocess_popen_read.py
subprocess_popen_write.py
subprocess_popen2.py
subprocess_popen3.py
subprocess_popen4.py
subprocess_pipes.py
repeater.py
interaction.py
signal_child.py
signal_parent.py
subprocess_signal_parent_shell.py
subprocess_signal_setpgrp.py
```

### Interacting with Another Command

所有先前的示例都假设有限量的交互。communicate（）方法读取所有输出并等待子进程在返回之前退出。当程序运行时，也可以逐步写入和读取Popen实例使用的各个管道句柄。从标准输入读取并写入标准输出的简单echo程序说明了这种技术。

脚本repeater.py在下一个示例中用作子进程。 它从stdin读取并将值写入stdout，一次一行，直到没有更多输入。 它还会在stderr启动和停止时向stderr写入一条消息，显示子进程的生命周期。

```python
#repeater.py
import sys

sys.stderr.write('repeater.py: starting\n')
sys.stderr.flush()

while True:
    next_line = sys.stdin.readline()
    sys.stderr.flush()
    if not next_line:
        break
    sys.stdout.write(next_line)
    sys.stdout.flush()

sys.stderr.write('repeater.py: exiting\n')
sys.stderr.flush()
```
下一个交互示例以不同方式使用Popen实例拥有的stdin和stdout文件句柄。在第一个示例中，将五个数字的序列写入进程的stdin，并在每次写入后读回下一行输出。在第二个示例中，写入相同的五个数字，但使用communicate（）一次读取所有输出。

```python
#interaction.py
import io
import subprocess

print('One line at a time:')
proc = subprocess.Popen(
    'python3 repeater.py',
    shell=True,
    stdin=subprocess.PIPE,
    stdout=subprocess.PIPE,
)
stdin = io.TextIOWrapper(
    proc.stdin,
    encoding='utf-8',
    line_buffering=True,  # send data on newline
)
stdout = io.TextIOWrapper(
    proc.stdout,
    encoding='utf-8',
)
for i in range(5):
    line = '{}\n'.format(i)
    stdin.write(line)
    output = stdout.readline()
    print(output.rstrip())
remainder = proc.communicate()[0].decode('utf-8')
print(remainder)

print()
print('All output at once:')
proc = subprocess.Popen(
    'python3 repeater.py',
    shell=True,
    stdin=subprocess.PIPE,
    stdout=subprocess.PIPE,
)
stdin = io.TextIOWrapper(
    proc.stdin,
    encoding='utf-8',
)
for i in range(5):
    line = '{}\n'.format(i)
    stdin.write(line)
stdin.flush()

output = proc.communicate()[0].decode('utf-8')
print(output)
```
“repeater.py：exiting”行出现在每个循环样式的输出中的不同点。

```
$ python3 -u interaction.py

One line at a time:
repeater.py: starting
0
1
2
3
4
repeater.py: exiting


All output at once:
repeater.py: starting
repeater.py: exiting
0
1
2
3
4
```

### Signaling Between Processes

os模块的进程管理示例包括使用os.fork（）和os.kill（）演示进程之间的信号。由于每个Popen实例都提供了一个带有子进程的进程ID的pid属性，因此可以使用subprocess执行类似的操作。下一个示例组合了两个脚本。该子进程为USR信号设置信号处理程序。

```python
#signal_child.py
import os
import signal
import time
import sys

pid = os.getpid()
received = False

def signal_usr1(signum, frame):
    "Callback invoked when a signal is received"
    global received
    received = True
    print('CHILD {:>6}: Received USR1'.format(pid))
    sys.stdout.flush()

print('CHILD {:>6}: Setting up signal handler'.format(pid))
sys.stdout.flush()
signal.signal(signal.SIGUSR1, signal_usr1)
print('CHILD {:>6}: Pausing to wait for signal'.format(pid))
sys.stdout.flush()
time.sleep(3)

if not received:
    print('CHILD {:>6}: Never received signal'.format(pid))
```
此脚本作为父进程运行。 它启动signal_child.py，然后发送USR1信号。

```python
#signal_parent.py
import os
import signal
import subprocess
import time
import sys

proc = subprocess.Popen(['python3', 'signal_child.py'])
print('PARENT      : Pausing before sending signal...')
sys.stdout.flush()
time.sleep(1)
print('PARENT      : Signaling child')
sys.stdout.flush()
os.kill(proc.pid, signal.SIGUSR1)
```
输出如下：

```
$ python3 signal_parent.py

PARENT      : Pausing before sending signal...
CHILD  26976: Setting up signal handler
CHILD  26976: Pausing to wait for signal
PARENT      : Signaling child
CHILD  26976: Received USR1
```

#### Process Groups / Sessions

如果由Popen创建的进程产生子进程，那些子进程将不会收到发送给父进程的任何信号。这意味着当对Popen使用shell参数时，很难通过发送SIGINT或SIGTERM来使shell中启动的命令终止。

```python
#subprocess_signal_parent_shell.py
import os
import signal
import subprocess
import tempfile
import time
import sys

script = '''#!/bin/sh
echo "Shell script in process $$"
set -x
python3 signal_child.py
'''
script_file = tempfile.NamedTemporaryFile('wt')
script_file.write(script)
script_file.flush()

proc = subprocess.Popen(['sh', script_file.name])
print('PARENT      : Pausing before signaling {}...'.format(proc.pid))
sys.stdout.flush()
time.sleep(1)
print('PARENT      : Signaling child {}'.format(proc.pid))
sys.stdout.flush()
os.kill(proc.pid, signal.SIGUSR1)
time.sleep(3)
```
用于发送信号的pid与等待信号的shell脚本的子进程的pid不匹配，因为在此示例中有三个独立的进程交互：

1. `subprocess_signal_parent_shell.py`程序
2. 运行主python程序创建的脚本的shell进程
3. `signal_child.py`程序

```
$ python3 subprocess_signal_parent_shell.py

PARENT      : Pausing before signaling 26984...
Shell script in process 26984
+ python3 signal_child.py
CHILD  26985: Setting up signal handler
CHILD  26985: Pausing to wait for signal
PARENT      : Signaling child 26984
CHILD  26985: Never received signal
```
要在不知道其进程ID的情况下向后代发送信号，请使用进程组关联子进程，以便可以一起接收信号。使用os.setpgrp（）创建进程组，该进程组将进程组id设置为当前进程的进程id。所有子进程都从其父进程继承其进程组，并且由于它只应在由Popen及其后代创建的shell中设置，因此不应在创建Popen实例的同一进程中调用os.setpgrp（）。相反，该函数( os.setpgrp() )作为preexec_fn参数传递给Popen，因此它在fork（）后的新进程内运行，然后使用exec（）运行shell。要发出整个进程组的信号，请使用os.killpg（）和Popen实例的pid值。

```python
#subprocess_signal_setpgrp.py
import os
import signal
import subprocess
import tempfile
import time
import sys

def show_setting_prgrp():
    print('Calling os.setpgrp() from {}'.format(os.getpid()))
    os.setpgrp()
    print('Process group is now {}'.format(os.getpgrp()))
    sys.stdout.flush()

script = '''#!/bin/sh
echo "Shell script in process $$"
set -x
python3 signal_child.py
'''
script_file = tempfile.NamedTemporaryFile('wt')
script_file.write(script)
script_file.flush()

proc = subprocess.Popen(
    ['sh', script_file.name],
    preexec_fn=show_setting_prgrp,
)
print('PARENT      : Pausing before signaling {}...'.format(proc.pid))
sys.stdout.flush()
time.sleep(1)
print('PARENT      : Signaling process group {}'.format(proc.pid))
sys.stdout.flush()
os.killpg(proc.pid, signal.SIGUSR1)
time.sleep(3)
```
事件的顺序是：

1. 父程序实例化Popen。
2. Popen实例fork了一个新进程。
3. 新进程运行os.setpgrp（）。
4. 新进程运行exec（）来启动shell。
5. shell运行shell脚本。
6. shell脚本再次fork，该进程执行Python。
7. Python运行signal_child.py。
8. 父程序使用shell的pid发信号给进程组。
9. shell和Python进程接收信号。
10. shell忽略了信号。
11. 运行signal_child.py的Python进程调用信号处理程序。

```
$ python3 subprocess_signal_setpgrp.py

Calling os.setpgrp() from 75636
Process group is now 75636
PARENT      : Pausing before signaling 75636...
Shell script in process 75636
+ python3 signal_child.py
CHILD  75637: Setting up signal handler
CHILD  75637: Pausing to wait for signal
PARENT      : Signaling process group 75636
CHILD  75637: Received USR1
```

## signal — Asynchronous System Events

目的：异步系统事件  
信号是一种操作系统功能，它提供了一种通知程序事件并使其异步处理的方法。它们可以由系统本身生成，也可以从一个进程发送到另一个进程。由于信号会中断程序的常规流程，因此如果在中间接收到信号，某些操作（尤其是I/O）可能会产生错误。

信号由整数标识，并在操作系统C头文件中定义。Python将适合于平台的信号公开为signal模块中的符号。本节中的示例使用SIGINT和SIGUSR1。两者通常都是针对所有Unix和类Unix系统定义的。

>注意  
使用Unix信号处理程序进行编程是一项非常重要的工作。这是一个介绍，并不包括在每个平台上成功使用信号所需的所有细节。不同版本的Unix有一定程度的标准化，但也有一些变化，所以如果遇到麻烦，请查阅操作系统文档。

### Receiving Signals

与其他形式的基于事件的编程一样，通过建立称为信号处理程序的回调函数来接收信号，该函数在信号发生时被调用。 信号处理程序的参数是信号编号和程序中被信号中断的点的堆栈帧。

```python
#signal_signal.py
import signal
import os
import time

def receive_signal(signum, stack):
    print('Received:', signum)

# Register signal handlers
signal.signal(signal.SIGUSR1, receive_signal)
signal.signal(signal.SIGUSR2, receive_signal)

# Print the process ID so it can be used with 'kill'
# to send this program signals.
print('My PID is:', os.getpid())

while True:
    print('Waiting...')
    time.sleep(3)
```
此示例脚本无限循环，每次暂停几秒钟。当信号到来时，sleep（）调用被中断，信号处理程序receive_signal打印信号编号。信号处理程序返回后，循环继续。  
使用os.kill（）或Unix命令行程序kill将信号发送到正在运行的程序。

```
$ python3 signal_signal.py

My PID is: 71387
Waiting...
Waiting...
Waiting...
Received: 30
Waiting...
Waiting...
Received: 31
Waiting...
Waiting...
Traceback (most recent call last):
  File "signal_signal.py", line 28, in <module>
    time.sleep(3)
KeyboardInterrupt
```
之前的输出是通过在一个窗口中运行signal_signal.py，然后在另一个窗口中运行如下命令生成的：

```
$ kill -USR1 $pid
$ kill -USR2 $pid
$ kill -INT $pid
```

### Retrieving Registered Handlers

要查看为信号注册了哪些信号处理程序，请使用getsignal（）。传递信号编号作为参数。返回值是已注册的处理程序，或特殊值`SIG_IGN`（如果信号被忽略），`SIG_DFL`（如果使用默认行为）或None之一（如果现有信号处理程序是从C注册的，而不是Python）。

```python
#signal_getsignal.py
import signal

def alarm_received(n, stack):
    return

signal.signal(signal.SIGALRM, alarm_received)

signals_to_names = {
    getattr(signal, n): n
    for n in dir(signal)
    if n.startswith('SIG') and '_' not in n
}

for s, name in sorted(signals_to_names.items()):
    handler = signal.getsignal(s)
    if handler is signal.SIG_DFL:
        handler = 'SIG_DFL'
    elif handler is signal.SIG_IGN:
        handler = 'SIG_IGN'
    print('{:<10} ({:2d}):'.format(name, s), handler)
```
同样，由于每个OS可能定义了不同的信号，因此其他系统上的输出可能会有所不同。 这是来自OS X：

```
$ python3 signal_getsignal.py

SIGHUP     ( 1): SIG_DFL
SIGINT     ( 2): <built-in function default_int_handler>
SIGQUIT    ( 3): SIG_DFL
SIGILL     ( 4): SIG_DFL
SIGTRAP    ( 5): SIG_DFL
SIGIOT     ( 6): SIG_DFL
SIGEMT     ( 7): SIG_DFL
SIGFPE     ( 8): SIG_DFL
SIGKILL    ( 9): None
SIGBUS     (10): SIG_DFL
SIGSEGV    (11): SIG_DFL
SIGSYS     (12): SIG_DFL
SIGPIPE    (13): SIG_IGN
SIGALRM    (14): <function alarm_received at 0x1019a6a60>
SIGTERM    (15): SIG_DFL
SIGURG     (16): SIG_DFL
SIGSTOP    (17): None
SIGTSTP    (18): SIG_DFL
SIGCONT    (19): SIG_DFL
SIGCHLD    (20): SIG_DFL
SIGTTIN    (21): SIG_DFL
SIGTTOU    (22): SIG_DFL
SIGIO      (23): SIG_DFL
SIGXCPU    (24): SIG_DFL
SIGXFSZ    (25): SIG_IGN
SIGVTALRM  (26): SIG_DFL
SIGPROF    (27): SIG_DFL
SIGWINCH   (28): SIG_DFL
SIGINFO    (29): SIG_DFL
SIGUSR1    (30): SIG_DFL
SIGUSR2    (31): SIG_DFL
```

### Sending Signals

从Python中发送信号的函数是os.kill（）。它的用法在os模块的[Createing Processes with os.fork()](https://pymotw.com/3/os/index.html#creating-processes-with-os-fork)部分中介绍。

### Alarms

警报是一种特殊的信号，程序要求操作系统在经过一段时间后通知它。正如os的标准模块文档指出的那样，这对于避免无限期地阻塞I/O操作或其他系统调用很有用。

```python
#signal_alarm.py
import signal
import time

def receive_alarm(signum, stack):
    print('Alarm :', time.ctime())

# Call receive_alarm in 2 seconds
signal.signal(signal.SIGALRM, receive_alarm)
signal.alarm(2)

print('Before:', time.ctime())
time.sleep(4)
print('After :', time.ctime())
```
在此示例中，对sleep（）的调用被中断，但在处理完信号后继续，因此sleep（）返回后打印的消息显示程序暂停至少与睡眠持续时间一样长。

```
$ python3 signal_alarm.py

Before: Sat Apr 22 14:48:57 2017
Alarm : Sat Apr 22 14:48:59 2017
After : Sat Apr 22 14:49:01 2017
```

### Ignoring Signals

要忽略信号，请将`SIG_IGN`注册为处理程序。 此脚本使用`SIG_IGN`替换`SIGINT`的缺省处理程序，并为SIGUSR1注册处理程序。 然后它使用signal.pause（）等待接收信号。

```python
#signal_ignore.py
import signal
import os
import time

def do_exit(sig, stack):
    raise SystemExit('Exiting')

signal.signal(signal.SIGINT, signal.SIG_IGN)
signal.signal(signal.SIGUSR1, do_exit)

print('My PID:', os.getpid())

signal.pause()
```
通常，SIGINT（当用户按下Ctrl-C时shell发送给程序的信号）会引发KeyboardInterrupt。此示例忽略SIGINT并在看到SIGUSR1时引发SystemExit。输出中的每个^C表示尝试使用Ctrl-C从终端终止脚本。从另一个终端使用kill -USR1 72598最终导致脚本退出。

```
$ python3 signal_ignore.py

My PID: 72598
^C^C^C^CExiting
```

### Signals and Threads

信号和线程通常不能很好地混合，因为只有进程的主线程才会收到信号。以下示例设置信号处理程序，在一个线程中等待信号，并从另一个线程发送信号。

```python
#signal_threads.py
import signal
import threading
import os
import time

def signal_handler(num, stack):
    print('Received signal {} in {}'.format(num, threading.currentThread().name))

signal.signal(signal.SIGUSR1, signal_handler)

def wait_for_signal():
    print('Waiting for signal in', threading.currentThread().name)
    signal.pause()
    print('Done waiting')

# Start a thread that will not receive the signal
receiver = threading.Thread(
    target=wait_for_signal,
    name='receiver',
)
receiver.start()
time.sleep(0.1)

def send_signal():
    print('Sending signal in', threading.currentThread().name)
    os.kill(os.getpid(), signal.SIGUSR1)

sender = threading.Thread(target=send_signal, name='sender')
sender.start()
sender.join()

# Wait for the thread to see the signal (not going to happen!)
print('Waiting for', receiver.name)
signal.alarm(2)
receiver.join()
```
信号处理程序都在主线程中注册，因为这是Python signal模块实现的要求，无论底层平台是否支持混合线程和信号。虽然接收器线程调用signal.pause（），但它不接收信号。 示例末尾附近的signal.alarm（2）调用会避免无限阻塞，因为接收器线程永远不会退出。

```
$ python3 signal_threads.py

Waiting for signal in receiver
Sending signal in sender
Received signal 30 in MainThread
Waiting for receiver
Alarm clock
```
虽然可以在任何线程中设置警报（alarm），但它们始终由主线程接收。

```python
#signal_threads_alarm.py
import signal
import time
import threading

def signal_handler(num, stack):
    print(time.ctime(), 'Alarm in', threading.currentThread().name)

signal.signal(signal.SIGALRM, signal_handler)

def use_alarm():
    t_name = threading.currentThread().name
    print(time.ctime(), 'Setting alarm in', t_name)
    signal.alarm(1)
    print(time.ctime(), 'Sleeping in', t_name)
    time.sleep(3)
    print(time.ctime(), 'Done with sleep in', t_name)

# Start a thread that will not receive the signal
alarm_thread = threading.Thread(
    target=use_alarm,
    name='alarm_thread',
)
alarm_thread.start()
time.sleep(0.1)

# Wait for the thread to see the signal (not going to happen!)
print(time.ctime(), 'Waiting for', alarm_thread.name)
alarm_thread.join()

print(time.ctime(), 'Exiting normally')
```
警报不会中止use_alarm（）中的sleep（）调用。

```
$ python3 signal_threads_alarm.py

Sat Apr 22 14:49:01 2017 Setting alarm in alarm_thread
Sat Apr 22 14:49:01 2017 Sleeping in alarm_thread
Sat Apr 22 14:49:01 2017 Waiting for alarm_thread
Sat Apr 22 14:49:02 2017 Alarm in MainThread
Sat Apr 22 14:49:04 2017 Done with sleep in alarm_thread
Sat Apr 22 14:49:04 2017 Exiting normally
```

## threading — Manage Concurrent Operations Within a Process

目的：管理多个执行线程。  
使用线程允许程序在同一进程空间中同时运行多个操作。

### Thread Objects

使用Thread的最简单方法是使用目标函数对其进行实例化，并调用start（）以使其开始工作。

```python
#threading_simple.py
import threading

def worker():
    """thread worker function"""
    print('Worker')

threads = []
for i in range(5):
    t = threading.Thread(target=worker)
    threads.append(t)
    t.start()
```
输出是五行，每行有“Worker”。

```
$ python3 threading_simple.py

Worker
Worker
Worker
Worker
Worker
```
能够生成一个线程并传递参数给它以告诉它要做什么工作是很有用的。任何类型的对象都可以作为参数传递给线程。此示例传递一个数字，然后线程将打印该数字。

```python
#threading_simpleargs.py
import threading

def worker(num):
    """thread worker function"""
    print('Worker: %s' % num)

threads = []
for i in range(5):
    t = threading.Thread(target=worker, args=(i,))
    threads.append(t)
    t.start()
```
整数参数现在包含在每个线程打印的消息中。

```
$ python3 threading_simpleargs.py

Worker: 0
Worker: 1
Worker: 2
Worker: 3
Worker: 4
```

### Determining the Current Thread

使用参数来识别或命名线程是麻烦且不必要的。每个Thread实例都有一个名称，其默认值可以在创建线程时更改。命名线程在服务器进程中很有用，其中多个服务线程处理不同的操作。

```python
#threading_names.py
import threading
import time

def worker():
    print(threading.current_thread().getName(), 'Starting')
    time.sleep(0.2)
    print(threading.current_thread().getName(), 'Exiting')

def my_service():
    print(threading.current_thread().getName(), 'Starting')
    time.sleep(0.3)
    print(threading.current_thread().getName(), 'Exiting')

t = threading.Thread(name='my_service', target=my_service)
w = threading.Thread(name='worker', target=worker)
w2 = threading.Thread(target=worker)  # use default name

w.start()
w2.start()
t.start()
```
调试输出包括当前线程的名称在每行上。线程名称列中带有“Thread-1”的行对应于未命名的线程w2。

```
$ python3 threading_names.py

worker Starting
Thread-1 Starting
my_service Starting
worker Exiting
Thread-1 Exiting
my_service Exiting
```
大多数程序不使用print进行调试。logging模块支持使用格式化程序代码`%(threadName)s`在每个日志消息中嵌入线程名称。在日志消息中包含线程名称可以追溯这些消息到他们的源头。

```python
#threading_names_log.py
import logging
import threading
import time

def worker():
    logging.debug('Starting')
    time.sleep(0.2)
    logging.debug('Exiting')

def my_service():
    logging.debug('Starting')
    time.sleep(0.3)
    logging.debug('Exiting')

logging.basicConfig(
    level=logging.DEBUG,
    format='[%(levelname)s] (%(threadName)-10s) %(message)s',
)

t = threading.Thread(name='my_service', target=my_service)
w = threading.Thread(name='worker', target=worker)
w2 = threading.Thread(target=worker)  # use default name

w.start()
w2.start()
t.start()
```
logging也是线程安全的，因此来自不同线程的消息在输出中保持不同。

```
$ python3 threading_names_log.py

[DEBUG] (worker    ) Starting
[DEBUG] (Thread-1  ) Starting
[DEBUG] (my_service) Starting
[DEBUG] (worker    ) Exiting
[DEBUG] (Thread-1  ) Exiting
[DEBUG] (my_service) Exiting
```

### Daemon vs. Non-Daemon Threads

到目前为止，示例程序隐式等待退出，直到所有线程完成其工作。有时，程序会生成一个线程为daemon，它的运行不会阻止主程序退出。对于可能没有简单方法来中断线程的服务，或者让线程在其工作中间死亡而不会丢失或损坏数据的服务（例如，生成“心跳”的线程，用于服务监控工具），使用daemon线程很有用。要将线程标记为daemon，请在构造时传递`daemon=True`或使用True调用其`set_daemon（）`方法。默认情况下，线程不是daemon。

```python
#threading_daemon.py
import threading
import time
import logging

def daemon():
    logging.debug('Starting')
    time.sleep(0.2)
    logging.debug('Exiting')

def non_daemon():
    logging.debug('Starting')
    logging.debug('Exiting')

logging.basicConfig(
    level=logging.DEBUG,
    format='(%(threadName)-10s) %(message)s',
)

d = threading.Thread(name='daemon', target=daemon, daemon=True)
t = threading.Thread(name='non-daemon', target=non_daemon)

d.start()
t.start()
```
输出不包括来自daemon线程的“Exiting”消息，因为所有non-daemon线程（包括主线程）在daemon线程从sleep（）调用中唤醒之前退出。

```
$ python3 threading_daemon.py

(daemon    ) Starting
(non-daemon) Starting
(non-daemon) Exiting
```
要等到daemon线程完成其工作，请使用join（）方法。

```python
#threading_daemon_join.py
import threading
import time
import logging

def daemon():
    logging.debug('Starting')
    time.sleep(0.2)
    logging.debug('Exiting')

def non_daemon():
    logging.debug('Starting')
    logging.debug('Exiting')

logging.basicConfig(
    level=logging.DEBUG,
    format='(%(threadName)-10s) %(message)s',
)

d = threading.Thread(name='daemon', target=daemon, daemon=True)
t = threading.Thread(name='non-daemon', target=non_daemon)

d.start()
t.start()

d.join()
t.join()
```
使用join（）等待daemon线程退出意味着它有机会生成其“Exiting”消息。

```
$ python3 threading_daemon_join.py

(daemon    ) Starting
(non-daemon) Starting
(non-daemon) Exiting
(daemon    ) Exiting
```
默认情况下，join（）无限期地阻塞。也可以传递一个浮点值，表示等待线程变为非活动状态的秒数。如果线程在超时期限内没有完成，则join（）仍会返回。

```python
#threading_daemon_join_timeout.py
import threading
import time
import logging

def daemon():
    logging.debug('Starting')
    time.sleep(0.2)
    logging.debug('Exiting')

def non_daemon():
    logging.debug('Starting')
    logging.debug('Exiting')

logging.basicConfig(
    level=logging.DEBUG,
    format='(%(threadName)-10s) %(message)s',
)

d = threading.Thread(name='daemon', target=daemon, daemon=True)
t = threading.Thread(name='non-daemon', target=non_daemon)

d.start()
t.start()

d.join(0.1)
print('d.isAlive()', d.isAlive())
t.join()
```
由于传递的超时时间小于daemon线程休眠的时间，因此在join（）返回后线程仍处于“活动状态”。

```
$ python3 threading_daemon_join_timeout.py

(daemon    ) Starting
(non-daemon) Starting
(non-daemon) Exiting
d.isAlive() True
```

### Enumerating All Threads

没有必要为所有daemon线程保留显式句柄，以确保它们在主进程退出之前已完成。enumerate（）返回活动Thread实例的列表。该列表包括当前线程，并且由于加入（join）当前线程会引入死锁情况，因此必须跳过它。

```python
#threading_enumerate.py
import random
import threading
import time
import logging

def worker():
    """thread worker function"""
    pause = random.randint(1, 5) / 10
    logging.debug('sleeping %0.2f', pause)
    time.sleep(pause)
    logging.debug('ending')

logging.basicConfig(
    level=logging.DEBUG,
    format='(%(threadName)-10s) %(message)s',
)

for i in range(3):
    t = threading.Thread(target=worker, daemon=True)
    t.start()

main_thread = threading.main_thread()
for t in threading.enumerate():
    if t is main_thread:
        continue
    logging.debug('joining %s', t.getName())
    t.join()
```
由于work休眠一段随机时间，因此该程序的输出可能会有所不同。

```
$ python3 threading_enumerate.py

(Thread-1  ) sleeping 0.20
(Thread-2  ) sleeping 0.30
(Thread-3  ) sleeping 0.40
(MainThread) joining Thread-1
(Thread-1  ) ending
(MainThread) joining Thread-3
(Thread-2  ) ending
(Thread-3  ) ending
(MainThread) joining Thread-2
```

### Subclassing Thread

在启动时，Thread执行一些基本的初始化，然后调用其run（）方法，该方法调用传递给构造函数的目标函数。要创建Thread的子类，请覆盖run（）以执行任何必要的操作。

```python
#threading_subclass.py
import threading
import logging

class MyThread(threading.Thread):
    def run(self):
        logging.debug('running')

logging.basicConfig(
    level=logging.DEBUG,
    format='(%(threadName)-10s) %(message)s',
)

for i in range(5):
    t = MyThread()
    t.start()
```
run（）的返回值被忽略。

```
$ python3 threading_subclass.py

(Thread-1  ) running
(Thread-2  ) running
(Thread-3  ) running
(Thread-4  ) running
(Thread-5  ) running
```
因为传递给Thread构造函数的args和kwargs值使用前缀为“__”的名称保存在私有变量中，所以不容易从子类访问它们。要将参数传递给自定义线程类型，请重新定义构造函数以将值保存在可在子类中看到的实例属性中。

```python
#threading_subclass_args.py
import threading
import logging

class MyThreadWithArgs(threading.Thread):
    def __init__(self, group=None, target=None, name=None,
                 args=(), kwargs=None, *, daemon=None):
        super().__init__(group=group, target=target, name=name,
                         daemon=daemon)
        self.args = args
        self.kwargs = kwargs

    def run(self):
        logging.debug('running with %s and %s', self.args, self.kwargs)

logging.basicConfig(
    level=logging.DEBUG,
    format='(%(threadName)-10s) %(message)s',
)

for i in range(5):
    t = MyThreadWithArgs(args=(i,), kwargs={'a': 'A', 'b': 'B'})
    t.start()
```
MyThreadWithArgs使用与Thread相同的API，但是另一个类可以轻松地更改构造函数方法，以获取与线程的目的更直接相关的更多或不同的参数，就像使用任何其他类一样。

```
$ python3 threading_subclass_args.py

(Thread-1  ) running with (0,) and {'b': 'B', 'a': 'A'}
(Thread-2  ) running with (1,) and {'b': 'B', 'a': 'A'}
(Thread-3  ) running with (2,) and {'b': 'B', 'a': 'A'}
(Thread-4  ) running with (3,) and {'b': 'B', 'a': 'A'}
(Thread-5  ) running with (4,) and {'b': 'B', 'a': 'A'}
```

### Timer Threads

Timer提供了一个例子说明要继承Thread的理由，改示例也包含在threading中。Timer在延迟一段时间后开始它的工作，并且可以在该延迟时间段内的任何时间点取消。

```python
#threading_timer.py
import threading
import time
import logging

def delayed():
    logging.debug('worker running')

logging.basicConfig(
    level=logging.DEBUG,
    format='(%(threadName)-10s) %(message)s',
)

t1 = threading.Timer(0.3, delayed)
t1.setName('t1')
t2 = threading.Timer(0.3, delayed)
t2.setName('t2')

logging.debug('starting timers')
t1.start()
t2.start()

logging.debug('waiting before canceling %s', t2.getName())
time.sleep(0.2)
logging.debug('canceling %s', t2.getName())
t2.cancel()
logging.debug('done')
```
此示例中的第二个计时器不会运行，并且第一个计时器似乎在主程序的其余部分完成后运行。由于它不是daemon线程，因此在完成主线程时会隐式join它。

```
$ python3 threading_timer.py

(MainThread) starting timers
(MainThread) waiting before canceling t2
(MainThread) canceling t2
(MainThread) done
(t1        ) worker running
```

### Signaling Between Threads

虽然使用多个线程的目的是同时运行单独的操作，但有时候能够在两个或多个线程中同步操作很重要。事件对象是一种在线程之间安全通信的简单方法。一个Event管理一个“调用者可以使用set（）和clear（）方法控制的”内部标志。其他线程可以使用wait（）暂停，直到标志被设置，在允许继续之前有效的阻止了进度。

```python
#threading_event.py
import logging
import threading
import time

def wait_for_event(e):
    """Wait for the event to be set before doing anything"""
    logging.debug('wait_for_event starting')
    event_is_set = e.wait()
    logging.debug('event set: %s', event_is_set)

def wait_for_event_timeout(e, t):
    """Wait t seconds and then timeout"""
    while not e.is_set():
        logging.debug('wait_for_event_timeout starting')
        event_is_set = e.wait(t)
        logging.debug('event set: %s', event_is_set)
        if event_is_set:
            logging.debug('processing event')
        else:
            logging.debug('doing other work')

logging.basicConfig(
    level=logging.DEBUG,
    format='(%(threadName)-10s) %(message)s',
)

e = threading.Event()
t1 = threading.Thread(name='block', target=wait_for_event, args=(e,), )
t1.start()

t2 = threading.Thread(name='nonblock', target=wait_for_event_timeout, args=(e, 2), )
t2.start()

logging.debug('Waiting before calling Event.set()')
time.sleep(0.3)
e.set()
logging.debug('Event is set')
```
wait（）方法接受一个参数，该参数表示在超时之前等待事件的秒数。它返回一个布尔值，指示是否设置了事件，因此调用者知道为什么返回wait（）。 is_set（）方法可以单独用于事件而不用担心阻塞。  
在此示例中，`wait_for_event_timeout（）`检查事件状态而不会无限期地阻塞。`wait_for_event（）`阻塞对wait（）的调用，在事件状态发生变化之前不会返回。

```
$ python3 threading_event.py

(block     ) wait_for_event starting
(nonblock  ) wait_for_event_timeout starting
(MainThread) Waiting before calling Event.set()
(MainThread) Event is set
(nonblock  ) event set: True
(nonblock  ) processing event
(block     ) event set: True
```

### Controlling Access to Resources

除了同步线程的操作之外，能够控制对共享资源的访问以防止损坏或丢失数据也很重要。Python的内置数据结构（列表，字典等）是线程安全的，因为具有操作这些结构的原子字节码（用于保护Python内部数据结构的全局解释器锁不会在更新之间释放）。用Python实现的其他数据结构，或整数和浮点数等更简单的类型，都没有这种保护。要防止同时访问一个对象，请使用Lock对象。

```python
#threading_lock.py
import logging
import random
import threading
import time

class Counter:

    def __init__(self, start=0):
        self.lock = threading.Lock()
        self.value = start

    def increment(self):
        logging.debug('Waiting for lock')
        self.lock.acquire()
        try:
            logging.debug('Acquired lock')
            self.value = self.value + 1
        finally:
            self.lock.release()

def worker(c):
    for i in range(2):
        pause = random.random()
        logging.debug('Sleeping %0.02f', pause)
        time.sleep(pause)
        c.increment()
    logging.debug('Done')

logging.basicConfig(
    level=logging.DEBUG,
    format='(%(threadName)-10s) %(message)s',
)

counter = Counter()
for i in range(2):
    t = threading.Thread(target=worker, args=(counter,))
    t.start()

logging.debug('Waiting for worker threads')
main_thread = threading.main_thread()
for t in threading.enumerate():
    if t is not main_thread:
        t.join()
logging.debug('Counter: %d', counter.value)
```
在此示例中，worker（）函数递增Counter实例，该实例维护Lock以防止两个线程同时更改其内部状态。如果未使用Lock，则可能会丢失对value属性的更改。

```
$ python3 threading_lock.py

(Thread-1  ) Sleeping 0.18
(Thread-2  ) Sleeping 0.93
(MainThread) Waiting for worker threads
(Thread-1  ) Waiting for lock
(Thread-1  ) Acquired lock
(Thread-1  ) Sleeping 0.11
(Thread-1  ) Waiting for lock
(Thread-1  ) Acquired lock
(Thread-1  ) Done
(Thread-2  ) Waiting for lock
(Thread-2  ) Acquired lock
(Thread-2  ) Sleeping 0.81
(Thread-2  ) Waiting for lock
(Thread-2  ) Acquired lock
(Thread-2  ) Done
(MainThread) Counter: 4
```
要查明另一个线程是否已获取锁而不阻塞当前线程，将blocking参数设为False传递给acquire（）。在下一个示例中，worker（）尝试分开的三次获取锁，并计算它必须执行的尝试次数。同时，`lock_holder（）`在保持和释放锁之间循环，每个状态中的短暂暂停用于模拟负载。

```python
#threading_lock_noblock.py
import logging
import threading
import time

def lock_holder(lock):
    logging.debug('Starting')
    while True:
        lock.acquire()
        try:
            logging.debug('Holding')
            time.sleep(0.5)
        finally:
            logging.debug('Not holding')
            lock.release()
        time.sleep(0.5)

def worker(lock):
    logging.debug('Starting')
    num_tries = 0
    num_acquires = 0
    while num_acquires < 3:
        time.sleep(0.5)
        logging.debug('Trying to acquire')
        have_it = lock.acquire(0)
        try:
            num_tries += 1
            if have_it:
                logging.debug('Iteration %d: Acquired', num_tries)
                num_acquires += 1
            else:
                logging.debug('Iteration %d: Not acquired', num_tries)
        finally:
            if have_it:
                lock.release()
    logging.debug('Done after %d iterations', num_tries)

logging.basicConfig(
    level=logging.DEBUG,
    format='(%(threadName)-10s) %(message)s',
)

lock = threading.Lock()

holder = threading.Thread(
    target=lock_holder,
    args=(lock,),
    name='LockHolder',
    daemon=True,
)
holder.start()

worker = threading.Thread(
    target=worker,
    args=(lock,),
    name='Worker',
)
worker.start()
```
worker（）需要三次以上的迭代才能获得三次锁定。

```
$ python3 threading_lock_noblock.py

(LockHolder) Starting
(LockHolder) Holding
(Worker    ) Starting
(LockHolder) Not holding
(Worker    ) Trying to acquire
(Worker    ) Iteration 1: Acquired
(LockHolder) Holding
(Worker    ) Trying to acquire
(Worker    ) Iteration 2: Not acquired
(LockHolder) Not holding
(Worker    ) Trying to acquire
(Worker    ) Iteration 3: Acquired
(LockHolder) Holding
(Worker    ) Trying to acquire
(Worker    ) Iteration 4: Not acquired
(LockHolder) Not holding
(Worker    ) Trying to acquire
(Worker    ) Iteration 5: Acquired
(Worker    ) Done after 5 iterations
```

#### Re-entrant Locks

即使是同一个线程，普通Lock对象也不能被获取超过一次。如果锁被同一调用链中的多个函数访问，则会引入不希望的副作用。

```python
#threading_lock_reacquire.py
import threading

lock = threading.Lock()

print('First try :', lock.acquire())
print('Second try:', lock.acquire(0))
```
在这种情况下，对acquire（）的第二次调用被赋予零超时以防止它被阻塞，因为第一次调用已获得锁定。

```
$ python3 threading_lock_reacquire.py

First try : True
Second try: False
```
在来自同一线程的分开的代码中需要“重新获取”锁的情况下，请使用RLock。

```python
#threading_rlock.py
import threading

lock = threading.RLock()

print('First try :', lock.acquire())
print('Second try:', lock.acquire(0))
```
对前一个示例中代码的唯一更改是将RLock替换为Lock。

```
$ python3 threading_rlock.py

First try : True
Second try: True
```

#### Locks as Context Managers

锁实现上下文管理器API并与with语句兼容。使用with消除了显式获取和释放锁的需要。

```python
#threading_lock_with.py
import threading
import logging

def worker_with(lock):
    with lock:
        logging.debug('Lock acquired via with')

def worker_no_with(lock):
    lock.acquire()
    try:
        logging.debug('Lock acquired directly')
    finally:
        lock.release()

logging.basicConfig(
    level=logging.DEBUG,
    format='(%(threadName)-10s) %(message)s',
)

lock = threading.Lock()
w = threading.Thread(target=worker_with, args=(lock,))
nw = threading.Thread(target=worker_no_with, args=(lock,))

w.start()
nw.start()
```
`worker_with（）`和`worker_no_with（）`这两个函数以等效的方式管理锁。

```
$ python3 threading_lock_with.py

(Thread-1  ) Lock acquired via with
(Thread-2  ) Lock acquired directly
```

### Synchronizing Threads

除了使用Events之外，另一种同步线程的方法是使用Condition对象。因为Condition使用Lock，所以它可以绑定到共享资源，允许多个线程等待资源更新。在此示例中，consumer（）线程在继续之前等待Condition被设置。producer（）线程负责设置条件并通知其他线程它们可以继续。

```python
#threading_condition.py
import logging
import threading
import time

def consumer(cond):
    """wait for the condition and use the resource"""
    logging.debug('Starting consumer thread')
    with cond:
        cond.wait()
        logging.debug('Resource is available to consumer')

def producer(cond):
    """set up the resource to be used by the consumer"""
    logging.debug('Starting producer thread')
    with cond:
        logging.debug('Making resource available')
        cond.notifyAll()

logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s (%(threadName)-2s) %(message)s',
)

condition = threading.Condition()
c1 = threading.Thread(name='c1', target=consumer, args=(condition,))
c2 = threading.Thread(name='c2', target=consumer, args=(condition,))
p  = threading.Thread(name='p',  target=producer, args=(condition,))

c1.start()
time.sleep(0.2)
c2.start()
time.sleep(0.2)
p.start()
```
线程使用with获取与Condition关联的锁。 显式地使用acquire（）和release（）方法也有效。

```
$ python3 threading_condition.py

2016-07-10 10:45:28,170 (c1) Starting consumer thread
2016-07-10 10:45:28,376 (c2) Starting consumer thread
2016-07-10 10:45:28,581 (p ) Starting producer thread
2016-07-10 10:45:28,581 (p ) Making resource available
2016-07-10 10:45:28,582 (c1) Resource is available to consumer
2016-07-10 10:45:28,582 (c2) Resource is available to consumer
```
Barriers（障碍）是另一种线程同步机制。Barrier建立一个控制点，所有参与的线程都会阻塞，直到所有参与的“各方”都已到达该点。它允许线程单独启动然后暂停，直到它们都准备好继续。

```python
#threading_barrier.py
import threading
import time

def worker(barrier):
    print(threading.current_thread().name,
          'waiting for barrier with {} others'.format(barrier.n_waiting))
    worker_id = barrier.wait()
    print(threading.current_thread().name, 'after barrier', worker_id)

NUM_THREADS = 3
barrier = threading.Barrier(NUM_THREADS)
threads = [
    threading.Thread(
        name='worker-%s' % i,
        target=worker,
        args=(barrier,),
    )
    for i in range(NUM_THREADS)
]

for t in threads:
    print(t.name, 'starting')
    t.start()
    time.sleep(0.1)

for t in threads:
    t.join()
```
在此示例中，Barrier配置为阻塞，直到三个线程都处于等待状态。满足条件时，所有线程同时通过控制点得到释放。wait（）的返回值表示正在被释放的编号，可用于限制某些线程采取清理共享资源等操作。

```
$ python3 threading_barrier.py

worker-0 starting
worker-0 waiting for barrier with 0 others
worker-1 starting
worker-1 waiting for barrier with 1 others
worker-2 starting
worker-2 waiting for barrier with 2 others
worker-2 after barrier 2
worker-0 after barrier 0
worker-1 after barrier 1
```
Barrier的abort（）方法导致所有等待中的线程接收BrokenBarrierError。当线程因阻塞在wait（）上而导致处理被停止时，则允许线程清理。

```python
#threading_barrier_abort.py
import threading
import time

def worker(barrier):
    print(threading.current_thread().name,
          'waiting for barrier with {} others'.format(barrier.n_waiting))
    try:
        worker_id = barrier.wait()
    except threading.BrokenBarrierError:
        print(threading.current_thread().name, 'aborting')
    else:
        print(threading.current_thread().name, 'after barrier', worker_id)

NUM_THREADS = 3
barrier = threading.Barrier(NUM_THREADS + 1)
threads = [
    threading.Thread(
        name='worker-%s' % i,
        target=worker,
        args=(barrier,),
    )
    for i in range(NUM_THREADS)
]

for t in threads:
    print(t.name, 'starting')
    t.start()
    time.sleep(0.1)

barrier.abort()

for t in threads:
    t.join()
```
此示例将Barrier配置为期望参与的线程比实际启动的更多，以便阻塞所有线程中的处理。abort（）调用在每个被阻塞的线程中引发异常。

```
$ python3 threading_barrier_abort.py

worker-0 starting
worker-0 waiting for barrier with 0 others
worker-1 starting
worker-1 waiting for barrier with 1 others
worker-2 starting
worker-2 waiting for barrier with 2 others
worker-0 aborting
worker-2 aborting
worker-1 aborting
```

### Limiting Concurrent Access to Resources

有时，允许多个worker同时访问一个资源是有用的，同时仍限制总数。例如，连接池可能支持固定数量的并发连接，或者网络应用程序可能支持固定数量的并发下载。Semaphore(信号量)是管理这些连接的一种方式。

```python
#threading_semaphore.py
import logging
import random
import threading
import time

class ActivePool:

    def __init__(self):
        super(ActivePool, self).__init__()
        self.active = []
        self.lock = threading.Lock()

    def makeActive(self, name):
        with self.lock:
            self.active.append(name)
            logging.debug('Running: %s', self.active)

    def makeInactive(self, name):
        with self.lock:
            self.active.remove(name)
            logging.debug('Running: %s', self.active)

def worker(s, pool):
    logging.debug('Waiting to join the pool')
    with s:
        name = threading.current_thread().getName()
        pool.makeActive(name)
        time.sleep(0.1)
        pool.makeInactive(name)

logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s (%(threadName)-2s) %(message)s',
)

pool = ActivePool()
s = threading.Semaphore(2)
for i in range(4):
    t = threading.Thread(
        target=worker,
        name=str(i),
        args=(s, pool),
    )
    t.start()
```
在此示例中，ActivePool类只是一种方便的方法来跟踪哪些线程能够在给定时刻运行。实际资源池会为新活动的线程分配连接或其他值，并在线程完成时回收该值。这里，它仅用于保存活动线程的名称，以显示最多两个并发运行。

```
$ python3 threading_semaphore.py

2016-07-10 10:45:29,398 (0 ) Waiting to join the pool
2016-07-10 10:45:29,398 (0 ) Running: ['0']
2016-07-10 10:45:29,399 (1 ) Waiting to join the pool
2016-07-10 10:45:29,399 (1 ) Running: ['0', '1']
2016-07-10 10:45:29,399 (2 ) Waiting to join the pool
2016-07-10 10:45:29,399 (3 ) Waiting to join the pool
2016-07-10 10:45:29,501 (1 ) Running: ['0']
2016-07-10 10:45:29,501 (0 ) Running: []
2016-07-10 10:45:29,502 (3 ) Running: ['3']
2016-07-10 10:45:29,502 (2 ) Running: ['3', '2']
2016-07-10 10:45:29,607 (3 ) Running: ['2']
2016-07-10 10:45:29,608 (2 ) Running: []
```

### Thread-specific Data

虽然某些资源需要被锁定，因此多个线程可以使用它们，但是其他资源需要受到保护，以便它们不受不拥有它们的线程的影响。local（）方法创建一个能够在单独的线程中“隐藏视图中的值”的对象。

```python
#threading_local.py
import random
import threading
import logging

def show_value(data):
    try:
        val = data.value
    except AttributeError:
        logging.debug('No value yet')
    else:
        logging.debug('value=%s', val)

def worker(data):
    show_value(data)
    data.value = random.randint(1, 100)
    show_value(data)

logging.basicConfig(
    level=logging.DEBUG,
    format='(%(threadName)-10s) %(message)s',
)

local_data = threading.local()
show_value(local_data)
local_data.value = 1000
show_value(local_data)

for i in range(2):
    t = threading.Thread(target=worker, args=(local_data,))
    t.start()
```
在该线程中设置属性之前，任何线程都不存在属性local_data.value。

```
$ python3 threading_local.py

(MainThread) No value yet
(MainThread) value=1000
(Thread-1  ) No value yet
(Thread-1  ) value=33
(Thread-2  ) No value yet
(Thread-2  ) value=74
```
要初始化设置以使所有线程都以相同的值开始，请使用子类并在`__init__（）`中设置属性。

```python
#threading_local_defaults.py
import random
import threading
import logging

def show_value(data):
    try:
        val = data.value
    except AttributeError:
        logging.debug('No value yet')
    else:
        logging.debug('value=%s', val)

def worker(data):
    show_value(data)
    data.value = random.randint(1, 100)
    show_value(data)

class MyLocal(threading.local):

    def __init__(self, value):
        super().__init__()
        logging.debug('Initializing %r', self)
        self.value = value

logging.basicConfig(
    level=logging.DEBUG,
    format='(%(threadName)-10s) %(message)s',
)

local_data = MyLocal(1000)
show_value(local_data)

for i in range(2):
    t = threading.Thread(target=worker, args=(local_data,))
    t.start()
```
`__init__（）`在同一个对象上调用（注意id（）值），在每个线程中设置一次默认值。

```
$ python3 threading_local_defaults.py

(MainThread) Initializing <__main__.MyLocal object at 0x101c6c288>
(MainThread) value=1000
(Thread-1  ) Initializing <__main__.MyLocal object at 0x101c6c288>
(Thread-1  ) value=1000
(Thread-1  ) value=18
(Thread-2  ) Initializing <__main__.MyLocal object at 0x101c6c288>
(Thread-2  ) value=1000
(Thread-2  ) value=77
```

## multiprocessing — Manage Processes Like Threads

目的：提供一个用于管理进程的API。  
multiprocessing模块包括一个用于在多个进程之间划分工作的API，基于用于threading的API。在某些情况下，multiprocessing是直接替代，代替threading来利用多个CPU核来避免与Python的全局解释器锁相关的计算瓶颈。  
由于相似性，此处的前几个示例是从threading示例中修改的。 稍后将介绍由multiprocessing提供但在threading中不可用的功能。

### multiprocessing Basics

产生第二个进程的最简单方法是使用目标函数实例化一个Process对象，并调用start（）让它开始工作。

```python
#multiprocessing_simple.py
import multiprocessing

def worker():
    """worker function"""
    print('Worker')

if __name__ == '__main__':
    jobs = []
    for i in range(5):
        p = multiprocessing.Process(target=worker)
        jobs.append(p)
        p.start()
```
输出包括打印五次的“Worker”一词，虽然它可能不是完全干净的，具体取决于执行顺序，因为每个进程都在竞争访问输出流。

```
$ python3 multiprocessing_simple.py

Worker
Worker
Worker
Worker
Worker
```
能够生成带有参数的进程以告诉它要做什么工作通常更有用。与threading不同，为了将参数传递给multiprocessing Process，参数必须能够使用pickle进行序列化。 此示例为每个worker传递要打印的数字。

```python
#multiprocessing_simpleargs.py
import multiprocessing

def worker(num):
    """thread worker function"""
    print('Worker:', num)

if __name__ == '__main__':
    jobs = []
    for i in range(5):
        p = multiprocessing.Process(target=worker, args=(i,))
        jobs.append(p)
        p.start()
```
整数参数现在包含在每个工作人员打印的消息中。

```
$ python3 multiprocessing_simpleargs.py

Worker: 1
Worker: 0
Worker: 2
Worker: 3
Worker: 4
```

### Importable Target Functions

threading和multiprocessing示例之间的一个区别是multiprocessing示例中使用的`__main__`有额外的保护。由于新进程的启动方式，子进程需要能够导入**包含目标函数**的脚本。在检查`__main__`时包装的应用程序的主要部分可确保在导入模块时不会在每个子进程中递归运行它（检查`__main__`及其后包装的代码）。另一种方法是从单独的脚本导入目标函数。例如，`multiprocessing_import_main.py`使用在第二个模块中定义的worker函数。

```python
#multiprocessing_import_main.py
import multiprocessing
import multiprocessing_import_worker

if __name__ == '__main__':
    jobs = []
    for i in range(5):
        p = multiprocessing.Process(target=multiprocessing_import_worker.worker,)
        jobs.append(p)
        p.start()
```
worker函数在`multiprocessing_import_worker.py`中定义。

```python
#multiprocessing_import_worker.py
def worker():
    """worker function"""
    print('Worker')
    return
```
调用主程序产生的输出类似于第一个例子。

```
$ python3 multiprocessing_import_main.py

Worker
Worker
Worker
Worker
Worker
```

### Determining the Current Process

传递参数来识别或命名进程是麻烦的，也是不必要的。 每个Process实例都有一个名称，其默认值可以在创建进程时更改。命名进程对于跟踪它们非常有用，尤其是在同时运行多种类型进程的应用程序中。

```python
#multiprocessing_names.py
import multiprocessing
import time

def worker():
    name = multiprocessing.current_process().name
    print(name, 'Starting')
    time.sleep(2)
    print(name, 'Exiting')

def my_service():
    name = multiprocessing.current_process().name
    print(name, 'Starting')
    time.sleep(3)
    print(name, 'Exiting')

if __name__ == '__main__':
    service = multiprocessing.Process(
        name='my_service',
        target=my_service,
    )
    worker_1 = multiprocessing.Process(
        name='worker 1',
        target=worker,
    )
    worker_2 = multiprocessing.Process(  # default name
        target=worker,
    )

    worker_1.start()
    worker_2.start()
    service.start()
```
调试输出包括每行上当前进程的名称。名称列中具有Process-3的行对应于未命名的进程worker_2。

```
$ python3 multiprocessing_names.py

worker 1 Starting
worker 1 Exiting
Process-3 Starting
Process-3 Exiting
my_service Starting
my_service Exiting
```

### Daemon Processes

默认情况下，在所有子进程退出之前，主程序不会退出。有些时候启动后台进程运行而不阻塞主程序退出是有用的，例如可能没有一种简单的方法来中断worker的服务，或者让它在工作过程中死亡而不会丢失或损坏数据的服务（例如，为服务监视工具生成“心跳”的任务）。  
要将进程标记为daemon，请将其daemon属性设置为True。默认进程不是daemon。

```python
#multiprocessing_daemon.py
import multiprocessing
import time
import sys

def daemon():
    p = multiprocessing.current_process()
    print('Starting:', p.name, p.pid)
    sys.stdout.flush()
    time.sleep(2)
    print('Exiting :', p.name, p.pid)
    sys.stdout.flush()

def non_daemon():
    p = multiprocessing.current_process()
    print('Starting:', p.name, p.pid)
    sys.stdout.flush()
    print('Exiting :', p.name, p.pid)
    sys.stdout.flush()

if __name__ == '__main__':
    d = multiprocessing.Process(
        name='daemon',
        target=daemon,
    )
    d.daemon = True

    n = multiprocessing.Process(
        name='non-daemon',
        target=non_daemon,
    )
    n.daemon = False

    d.start()
    time.sleep(1)
    n.start()
```
输出不包括来自daemon进程的“Exiting”消息，因为所有非守护进程（包括主程序）在守护进程从其2s休眠状态唤醒之前退出。

```
$ python3 multiprocessing_daemon.py

Starting: daemon 41838
Starting: non-daemon 41841
Exiting : non-daemon 41841
```
守护进程在主程序退出之前自动终止，这避免了孤儿进程的运行。这可以通过查找程序运行时打印的进程id值，然后使用ps等命令检查该进程进行验证。

### Waiting for Processes

要等到进程完成其工作并退出，请使用join（）方法。

```python
#multiprocessing_daemon_join.py
import multiprocessing
import time
import sys

def daemon():
    name = multiprocessing.current_process().name
    print('Starting:', name)
    time.sleep(2)
    print('Exiting :', name)

def non_daemon():
    name = multiprocessing.current_process().name
    print('Starting:', name)
    print('Exiting :', name)

if __name__ == '__main__':
    d = multiprocessing.Process(
        name='daemon',
        target=daemon,
    )
    d.daemon = True

    n = multiprocessing.Process(
        name='non-daemon',
        target=non_daemon,
    )
    n.daemon = False

    d.start()
    time.sleep(1)
    n.start()

    d.join()
    n.join()
```
由于主进程使用join（）等待守护进程退出，因此这次打印出“Exiting”消息。

```
$ python3 multiprocessing_daemon_join.py

Starting: non-daemon
Exiting : non-daemon
Starting: daemon
Exiting : daemon
```
默认情况下，join（）无限期地阻塞。也可以传递一个超时参数（一个浮点数表示等待进程变为非活动状态的秒数）。如果进程未在超时期限内完成，则join（）仍会返回。

```python
#multiprocessing_daemon_join_timeout.py
import multiprocessing
import time
import sys

def daemon():
    name = multiprocessing.current_process().name
    print('Starting:', name)
    time.sleep(2)
    print('Exiting :', name)

def non_daemon():
    name = multiprocessing.current_process().name
    print('Starting:', name)
    print('Exiting :', name)

if __name__ == '__main__':
    d = multiprocessing.Process(
        name='daemon',
        target=daemon,
    )
    d.daemon = True

    n = multiprocessing.Process(
        name='non-daemon',
        target=non_daemon,
    )
    n.daemon = False

    d.start()
    n.start()

    d.join(1)
    print('d.is_alive()', d.is_alive())
    n.join()
```
由于传递的超时时间小于守护进程休眠的时间，因此在join（）返回后进程仍处于“活动状态”。

```
$ python3 multiprocessing_daemon_join_timeout.py

Starting: non-daemon
Exiting : non-daemon
d.is_alive() True
```

### Terminating Processes

虽然最好使用“用信号通知进程应该退出”这样的毒丸（poison pill）方法（请参阅本章后面的“Passing Messages to Processes”），但是如果进程出现挂起或死锁，那么强制终止它是有用的。在进程对象上调用terminate（）会终止其子进程。

```python
#multiprocessing_terminate.py
import multiprocessing
import time

def slow_worker():
    print('Starting worker')
    time.sleep(0.1)
    print('Finished worker')

if __name__ == '__main__':
    p = multiprocessing.Process(target=slow_worker)
    print('BEFORE:', p, p.is_alive())

    p.start()
    print('DURING:', p, p.is_alive())

    p.terminate()
    print('TERMINATED:', p, p.is_alive())

    p.join()
    print('JOINED:', p, p.is_alive())
```
>注意  
在终止它之后join（）进程非常重要，以便为进程管理代码提供时间来更新对象的状态以反映终止。

```
$ python3 multiprocessing_terminate.py

BEFORE: <Process(Process-1, initial)> False
DURING: <Process(Process-1, started)> True
TERMINATED: <Process(Process-1, started)> True
JOINED: <Process(Process-1, stopped[SIGTERM])> False
```

### Process Exit Status

可以通过exitcode属性访问进程退出时生成的状态码。允许的范围列于下表中。  
Multiprocessing Exit Codes

Exit Code | Meaning
:---      | :---
`== 0`	  | 没有错误产生
`> 0`	  | 该进程出错，并以该状态码退出
`< 0`	  | 这个进程被（-1*exitcode）信号杀死

```python
#multiprocessing_exitcode.py
import multiprocessing
import sys
import time

def exit_error():
    sys.exit(1)

def exit_ok():
    return

def return_value():
    return 1

def raises():
    raise RuntimeError('There was an error!')

def terminated():
    time.sleep(3)

if __name__ == '__main__':
    jobs = []
    funcs = [
        exit_error,
        exit_ok,
        return_value,
        raises,
        terminated,
    ]
    for f in funcs:
        print('Starting process for', f.__name__)
        j = multiprocessing.Process(target=f, name=f.__name__)
        jobs.append(j)
        j.start()

    jobs[-1].terminate()

    for j in jobs:
        j.join()
        print('{:>15}.exitcode = {}'.format(j.name, j.exitcode))
```
引发异常的进程自动获得1的exitcode。

```
$ python3 multiprocessing_exitcode.py

Starting process for exit_error
Starting process for exit_ok
Starting process for return_value
Starting process for raises
Starting process for terminated
Process raises:
Traceback (most recent call last):
  File ".../lib/python3.6/multiprocessing/process.py", line 258,
in _bootstrap
    self.run()
  File ".../lib/python3.6/multiprocessing/process.py", line 93,
in run
    self._target(*self._args, **self._kwargs)
  File "multiprocessing_exitcode.py", line 28, in raises
    raise RuntimeError('There was an error!')
RuntimeError: There was an error!
     exit_error.exitcode = 1
        exit_ok.exitcode = 0
   return_value.exitcode = 0
         raises.exitcode = 1
     terminated.exitcode = -15
```

### Logging

在调试并发问题时，能访问multiprocessing提供的对象的内部结构会很有用。有一个方便的模块级函数来启用日志记录名为`log_to_stderr（）`。它使用logging设置记录器对象并添加处理程序，以便将日志消息发送到标准错误通道。

```python
#multiprocessing_log_to_stderr.py
import multiprocessing
import logging
import sys

def worker():
    print('Doing some work')
    sys.stdout.flush()

if __name__ == '__main__':
    multiprocessing.log_to_stderr(logging.DEBUG)
    p = multiprocessing.Process(target=worker)
    p.start()
    p.join()
```
默认情况下，日志记录级别设置为NOTSET，因此不会生成任何消息。传递不同的级别以将记录器初始化为所需的详细程度。

```
$ python3 multiprocessing_log_to_stderr.py

[INFO/Process-1] child process calling self.run() Doing some work
[INFO/Process-1] process shutting down
[DEBUG/Process-1] running all "atexit" finalizers with priority >= 0
[DEBUG/Process-1] running the remaining "atexit" finalizers
[INFO/Process-1] process exiting with exitcode 0
[INFO/MainProcess] process shutting down
[DEBUG/MainProcess] running all "atexit" finalizers with priority >= 0
[DEBUG/MainProcess] running the remaining "atexit" finalizers
```
要直接操作记录器（更改其级别设置或添加处理程序），请使用get_logger（）。

```python
#multiprocessing_get_logger.py
import multiprocessing
import logging
import sys

def worker():
    print('Doing some work')
    sys.stdout.flush()

if __name__ == '__main__':
    multiprocessing.log_to_stderr()
    logger = multiprocessing.get_logger()
    logger.setLevel(logging.INFO)
    p = multiprocessing.Process(target=worker)
    p.start()
    p.join()
```
还可以使用名称“multiprocessing” 通过logging配置文件API配置记录器。

```
$ python3 multiprocessing_get_logger.py

[INFO/Process-1] child process calling self.run() Doing some work
[INFO/Process-1] process shutting down
[INFO/Process-1] process exiting with exitcode 0
[INFO/MainProcess] process shutting down
```

### Subclassing Process

虽然在单独的进程中启动作业的最简单方法是使用Process并传递目标函数，但也可以使用自定义子类。

```python
#multiprocessing_subclass.py
import multiprocessing

class Worker(multiprocessing.Process):

    def run(self):
        print('In {}'.format(self.name))
        return

if __name__ == '__main__':
    jobs = []
    for i in range(5):
        p = Worker()
        jobs.append(p)
        p.start()
    for j in jobs:
        j.join()
```
派生类应该重写`run（）`来完成它的工作。

```
$ python3 multiprocessing_subclass.py

In Worker-1
In Worker-3
In Worker-2
In Worker-4
In Worker-5
```

### Passing Messages to Processes

与线程一样，多个进程的常见使用模式是将作业划分为多个工作并行运行。有效使用多个进程通常需要在它们之间进行一些通信，以便可以划分工作并汇总结果。使用multiprocessing在进程之间进行通信的一种简单方法是使用Queue来回传递消息。任何可以使用pickle序列化的对象都可以通过Queue传递。

```python
#multiprocessing_queue.py
import multiprocessing

class MyFancyClass:

    def __init__(self, name):
        self.name = name

    def do_something(self):
        proc_name = multiprocessing.current_process().name
        print('Doing something fancy in {} for {}!'.format(proc_name, self.name))

def worker(q):
    obj = q.get()
    obj.do_something()

if __name__ == '__main__':
    queue = multiprocessing.Queue()

    p = multiprocessing.Process(target=worker, args=(queue,))
    p.start()

    queue.put(MyFancyClass('Fancy Dan'))

    # Wait for the worker to finish
    queue.close()
    queue.join_thread()
    p.join()
```
这个简短的示例仅将单个消息传递给单个工作程序，然后主进程等待工作程序完成。

```
$ python3 multiprocessing_queue.py

Doing something fancy in Process-1 for Fancy Dan!
```

更复杂的示例显示了如何管理从JoinableQueue消费数据并将结果传递回父进程的多个worker。毒丸(poison pill)技术用于停止workers。设置实际任务后，主程序会将每个worker的一个“stop”值添加到作业队列中。当worker遇到特殊值时，它会跳出其处理循环。主进程使用任务队列的join（）方法在处理结果之前等待所有任务完成。

```python
#multiprocessing_producer_consumer.py
import multiprocessing
import time

class Consumer(multiprocessing.Process):

    def __init__(self, task_queue, result_queue):
        multiprocessing.Process.__init__(self)
        self.task_queue = task_queue
        self.result_queue = result_queue

    def run(self):
        proc_name = self.name
        while True:
            next_task = self.task_queue.get()
            if next_task is None:
                # Poison pill means shutdown
                print('{}: Exiting'.format(proc_name))
                self.task_queue.task_done()
                break
            print('{}: {}'.format(proc_name, next_task))
            answer = next_task()
            self.task_queue.task_done()
            self.result_queue.put(answer)

class Task:

    def __init__(self, a, b):
        self.a = a
        self.b = b

    def __call__(self):
        time.sleep(0.1)  # pretend to take time to do the work
        return '{self.a} * {self.b} = {product}'.format(
            self=self, product=self.a * self.b)

    def __str__(self):
        return '{self.a} * {self.b}'.format(self=self)

if __name__ == '__main__':
    # Establish communication queues
    tasks = multiprocessing.JoinableQueue()
    results = multiprocessing.Queue()

    # Start consumers
    num_consumers = multiprocessing.cpu_count() * 2
    print('Creating {} consumers'.format(num_consumers))
    consumers = [
        Consumer(tasks, results)
        for i in range(num_consumers)
    ]
    for w in consumers:
        w.start()

    # Enqueue jobs
    num_jobs = 10
    for i in range(num_jobs):
        tasks.put(Task(i, i))

    # Add a poison pill for each consumer
    for i in range(num_consumers):
        tasks.put(None)

    # Wait for all of the tasks to finish
    tasks.join()

    # Start printing results
    while num_jobs:
        result = results.get()
        print('Result:', result)
        num_jobs -= 1
```
尽管作业按顺序进入队列，但它们的执行是并行化的，因此无法保证它们的完成顺序。

```
$ python3 -u multiprocessing_producer_consumer.py

Creating 8 consumers
Consumer-1: 0 * 0
Consumer-2: 1 * 1
Consumer-3: 2 * 2
Consumer-4: 3 * 3
Consumer-5: 4 * 4
Consumer-6: 5 * 5
Consumer-7: 6 * 6
Consumer-8: 7 * 7
Consumer-3: 8 * 8
Consumer-7: 9 * 9
Consumer-4: Exiting
Consumer-1: Exiting
Consumer-2: Exiting
Consumer-5: Exiting
Consumer-6: Exiting
Consumer-8: Exiting
Consumer-7: Exiting
Consumer-3: Exiting
Result: 6 * 6 = 36
Result: 2 * 2 = 4
Result: 3 * 3 = 9
Result: 0 * 0 = 0
Result: 1 * 1 = 1
Result: 7 * 7 = 49
Result: 4 * 4 = 16
Result: 5 * 5 = 25
Result: 8 * 8 = 64
Result: 9 * 9 = 81
```

### Signaling between Processes

Event类提供了一种在进程之间传递状态信息的简单方法。事件可以在设置状态和未设置状态之间切换。使用事件对象的用户可以使用可选的超时值等待它从未设置状态更改为设置状态。

```python
#multiprocessing_event.py
import multiprocessing
import time

def wait_for_event(e):
    """Wait for the event to be set before doing anything"""
    print('wait_for_event: starting')
    e.wait()
    print('wait_for_event: e.is_set()->', e.is_set())

def wait_for_event_timeout(e, t):
    """Wait t seconds and then timeout"""
    print('wait_for_event_timeout: starting')
    e.wait(t)
    print('wait_for_event_timeout: e.is_set()->', e.is_set())

if __name__ == '__main__':
    e = multiprocessing.Event()
    w1 = multiprocessing.Process(
        name='block',
        target=wait_for_event,
        args=(e,),
    )
    w1.start()

    w2 = multiprocessing.Process(
        name='nonblock',
        target=wait_for_event_timeout,
        args=(e, 2),
    )
    w2.start()

    print('main: waiting before calling Event.set()')
    time.sleep(3)
    e.set()
    print('main: event is set')
```
当wait（）超时时，它不会返回错误。调用者负责使用is_set（）检查事件的状态。

```
$ python3 -u multiprocessing_event.py

main: waiting before calling Event.set()
wait_for_event: starting
wait_for_event_timeout: starting
wait_for_event_timeout: e.is_set()-> False
main: event is set
wait_for_event: e.is_set()-> True
```

### Controlling Access to Resources

在需要在多个进程之间共享单个资源的情况下，可以使用Lock来避免冲突访问。

```python
#multiprocessing_lock.py
import multiprocessing
import sys

def worker_with(lock, stream):
    with lock:
        stream.write('Lock acquired via with\n')

def worker_no_with(lock, stream):
    lock.acquire()
    try:
        stream.write('Lock acquired directly\n')
    finally:
        lock.release()

lock = multiprocessing.Lock()
w = multiprocessing.Process(
    target=worker_with,
    args=(lock, sys.stdout),
)
nw = multiprocessing.Process(
    target=worker_no_with,
    args=(lock, sys.stdout),
)

w.start()
nw.start()

w.join()
nw.join()
```
在此示例中，如果两个进程不使用锁同步它们对输出流的访问，则打印到控制台的消息可能混杂在一起。

```
$ python3 multiprocessing_lock.py

Lock acquired via with
Lock acquired directly
```

### Synchronizing Operations

Condition对象可用于同步工作流的各个部分，以便某些部分并行运行，但其他部分顺序运行，即使它们位于不同的进程中。

```python
#multiprocessing_condition.py
import multiprocessing
import time

def stage_1(cond):
    """perform first stage of work,
    then notify stage_2 to continue
    """
    name = multiprocessing.current_process().name
    print('Starting', name)
    with cond:
        print('{} done and ready for stage 2'.format(name))
        cond.notify_all()

def stage_2(cond):
    """wait for the condition telling us stage_1 is done"""
    name = multiprocessing.current_process().name
    print('Starting', name)
    with cond:
        cond.wait()
        print('{} running'.format(name))

if __name__ == '__main__':
    condition = multiprocessing.Condition()
    s1 = multiprocessing.Process(name='s1',
                                 target=stage_1,
                                 args=(condition,))
    s2_clients = [
        multiprocessing.Process(
            name='stage_2[{}]'.format(i),
            target=stage_2,
            args=(condition,),
        )
        for i in range(1, 3)
    ]

    for c in s2_clients:
        c.start()
        time.sleep(1)
    s1.start()

    s1.join()
    for c in s2_clients:
        c.join()
```
在此示例中，两个进程并行运行作业的第二个阶段，但仅在第一个阶段完成后运行。

```
$ python3 -u multiprocessing_condition.py

Starting stage_2[1]
Starting stage_2[2]
Starting s1
s1 done and ready for stage 2
stage_2[1] running
stage_2[2] running
```

### Controlling Concurrent Access to Resources

有时，允许多个worker同时访问一个资源是有用的，同时仍限制总数。例如，连接池可能支持固定数量的并发连接，或者网络应用程序可能支持固定数量的并发下载。Semaphore(信号量)是管理这些连接的一种方式。

```python
#multiprocessing_semaphore.py
import random
import multiprocessing
import time

class ActivePool:

    def __init__(self):
        super(ActivePool, self).__init__()
        self.mgr = multiprocessing.Manager()
        self.active = self.mgr.list()
        self.lock = multiprocessing.Lock()

    def makeActive(self, name):
        with self.lock:
            self.active.append(name)

    def makeInactive(self, name):
        with self.lock:
            self.active.remove(name)

    def __str__(self):
        with self.lock:
            return str(self.active)

def worker(s, pool):
    name = multiprocessing.current_process().name
    with s:
        pool.makeActive(name)
        print('Activating {} now running {}'.format(name, pool))
        time.sleep(random.random())
        pool.makeInactive(name)

if __name__ == '__main__':
    pool = ActivePool()
    s = multiprocessing.Semaphore(3)
    jobs = [
        multiprocessing.Process(
            target=worker,
            name=str(i),
            args=(s, pool),
        )
        for i in range(10)
    ]

    for j in jobs:
        j.start()

    while True:
        alive = 0
        for j in jobs:
            if j.is_alive():
                alive += 1
                j.join(timeout=0.1)
                print('Now running {}'.format(pool))
        if alive == 0:
            # all done
            break
```
在此示例中，ActivePool类只是用于跟踪在给定时刻正在运行的进程的便捷方式。实际资源池可能会为新活动的进程分配连接或其他值，并在任务完成时回收该值。这里，池只用于保存活动进程的名称，以显示只有三个并发运行。

```
$ python3 -u multiprocessing_semaphore.py

Activating 0 now running ['0', '1', '2']
Activating 1 now running ['0', '1', '2']
Activating 2 now running ['0', '1', '2']
Now running ['0', '1', '2']
Now running ['0', '1', '2']
Now running ['0', '1', '2']
Now running ['0', '1', '2']
Activating 3 now running ['0', '1', '3']
Activating 4 now running ['1', '3', '4']
Activating 6 now running ['1', '4', '6']
Now running ['1', '4', '6']
Now running ['1', '4', '6']
Activating 5 now running ['1', '4', '5']
Now running ['1', '4', '5']
Now running ['1', '4', '5']
Now running ['1', '4', '5']
Activating 8 now running ['4', '5', '8']
Now running ['4', '5', '8']
Now running ['4', '5', '8']
Now running ['4', '5', '8']
Now running ['4', '5', '8']
Now running ['4', '5', '8']
Activating 7 now running ['5', '8', '7']
Now running ['5', '8', '7']
Activating 9 now running ['8', '7', '9']
Now running ['8', '7', '9']
Now running ['8', '9']
Now running ['8', '9']
Now running ['9']
Now running ['9']
Now running ['9']
Now running ['9']
Now running []
```

### Managing Shared State

在前面的示例中，活动进程列表通过Manager创建的特殊类型的列表对象在ActivePool实例中集中维护。Manager负责协调所有用户之间的共享信息状态。

```python
#multiprocessing_manager_dict.py
import multiprocessing
import pprint

def worker(d, key, value):
    d[key] = value

if __name__ == '__main__':
    mgr = multiprocessing.Manager()
    d = mgr.dict()
    jobs = [
        multiprocessing.Process(
            target=worker,
            args=(d, i, i * 2),
        )
        for i in range(10)
    ]
    for j in jobs:
        j.start()
    for j in jobs:
        j.join()
    print('Results:', d)
```
通过管理器创建列表，它将被共享，并且更新可以被所有进程看到。字典也受支持。

```
$ python3 multiprocessing_manager_dict.py

Results: {0: 0, 1: 2, 2: 4, 3: 6, 4: 8, 5: 10, 6: 12, 7: 14, 8: 16, 9: 18}
```

### Shared Namespaces

除字典和列表外，Manager还可以创建共享Namespace（命名空间）。

```python
#multiprocessing_namespaces.py
import multiprocessing

def producer(ns, event):
    ns.value = 'This is the value'
    event.set()

def consumer(ns, event):
    try:
        print('Before event: {}'.format(ns.value))
    except Exception as err:
        print('Before event, error:', str(err))
    event.wait()
    print('After event:', ns.value)

if __name__ == '__main__':
    mgr = multiprocessing.Manager()
    namespace = mgr.Namespace()
    event = multiprocessing.Event()
    p = multiprocessing.Process(
        target=producer,
        args=(namespace, event),
    )
    c = multiprocessing.Process(
        target=consumer,
        args=(namespace, event),
    )

    c.start()
    p.start()

    c.join()
    p.join()
```
添加到命名空间的任何命名值对于接收命名空间实例的所有客户端都是可见的。

```
$ python3 multiprocessing_namespaces.py

Before event, error: 'Namespace' object has no attribute 'value'
After event: This is the value
```
重要的是要知道命名空间中可变值内容的更新不会自动传播。

```python
#multiprocessing_namespaces_mutable.py
import multiprocessing

def producer(ns, event):
    # DOES NOT UPDATE GLOBAL VALUE!
    ns.my_list.append('This is the value')
    event.set()

def consumer(ns, event):
    print('Before event:', ns.my_list)
    event.wait()
    print('After event :', ns.my_list)

if __name__ == '__main__':
    mgr = multiprocessing.Manager()
    namespace = mgr.Namespace()
    namespace.my_list = []

    event = multiprocessing.Event()
    p = multiprocessing.Process(
        target=producer,
        args=(namespace, event),
    )
    c = multiprocessing.Process(
        target=consumer,
        args=(namespace, event),
    )

    c.start()
    p.start()

    c.join()
    p.join()
```
要更新列表，请再次将其附加到命名空间对象。

```
$ python3 multiprocessing_namespaces_mutable.py

Before event: []
After event : []
```

### Process Pools

Pool类可用于管理固定数量的worker，以处理“要完成的工作可以分解并在worker之间独立分配的”简单情况。收集作业的返回值并作为列表返回。pool参数包括进程数和启动任务进程时要运行的函数（每个子进程调用一次）。

```python
#multiprocessing_pool.py
import multiprocessing

def do_calculation(data):
    return data * 2

def start_process():
    print('Starting', multiprocessing.current_process().name)

if __name__ == '__main__':
    inputs = list(range(10))
    print('Input   :', inputs)

    builtin_outputs = map(do_calculation, inputs)
    print('Built-in:', builtin_outputs)

    pool_size = multiprocessing.cpu_count() * 2
    pool = multiprocessing.Pool(
        processes=pool_size,
        initializer=start_process,
    )
    pool_outputs = pool.map(do_calculation, inputs)
    pool.close()  # no more tasks
    pool.join()  # wrap up current tasks

    print('Pool    :', pool_outputs)
```
map（）方法的结果在功能上等同于内置map（），除了每个任务并行运行。由于池并行处理其输入，因此可以使用close（）和join（）来将主进程与任务进程同步，以确保正确清理。

```
$ python3 multiprocessing_pool.py

Input   : [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
Built-in: <map object at 0x1007b2be0>
Starting ForkPoolWorker-3
Starting ForkPoolWorker-4
Starting ForkPoolWorker-5
Starting ForkPoolWorker-6
Starting ForkPoolWorker-1
Starting ForkPoolWorker-7
Starting ForkPoolWorker-2
Starting ForkPoolWorker-8
Pool    : [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]
```
默认情况下，Pool会创建固定数量的工作进程并将作业传递给它们，直到没有其他作业为止。设置maxtasksperchild参数会告诉池在完成一些任务后重新启动工作进程，从而防止长时间运行的工作进程消耗更多的系统资源。

```python
#multiprocessing_pool_maxtasksperchild.py
import multiprocessing

def do_calculation(data):
    return data * 2

def start_process():
    print('Starting', multiprocessing.current_process().name)

if __name__ == '__main__':
    inputs = list(range(10))
    print('Input   :', inputs)

    builtin_outputs = map(do_calculation, inputs)
    print('Built-in:', builtin_outputs)

    pool_size = multiprocessing.cpu_count() * 2
    pool = multiprocessing.Pool(
        processes=pool_size,
        initializer=start_process,
        maxtasksperchild=2,
    )
    pool_outputs = pool.map(do_calculation, inputs)
    pool.close()  # no more tasks
    pool.join()  # wrap up current tasks

    print('Pool    :', pool_outputs)
```
即使没有更多工作，池也会在完成分配给他的任务后重新启动worker。在此输出中，会创建8个工作进程，即使只有10个任务，并且每个worker可以一次完成其中两个任务。

```
$ python3 multiprocessing_pool_maxtasksperchild.py

Input   : [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
Built-in: <map object at 0x1007b21d0>
Starting ForkPoolWorker-1
Starting ForkPoolWorker-2
Starting ForkPoolWorker-4
Starting ForkPoolWorker-5
Starting ForkPoolWorker-6
Starting ForkPoolWorker-3
Starting ForkPoolWorker-7
Starting ForkPoolWorker-8
Pool    : [0, 2, 4, 6, 8, 10, 12, 14, 16, 18]
```

### Implementing MapReduce

Pool类可用于创建简单的单服务器MapReduce实现。虽然它没有给出分布式处理的全部好处，但它确实说明了将一些问题分解为可分配的工作单元是多么容易。  
在基于MapReduce的系统中，输入数据被分解为块以供不同的worker实例处理。使用简单的变换将每个输入数据块map到中间状态。然后将中间数据收集在一起并基于键值进行分区，以使所有相关值在一起。最后，分区数据被reduce到结果集。

```python
#multiprocessing_mapreduce.py
import collections
import itertools
import multiprocessing

class SimpleMapReduce:

    def __init__(self, map_func, reduce_func, num_workers=None):
        """
        map_func

          Function to map inputs to intermediate data. Takes as
          argument one input value and returns a tuple with the
          key and a value to be reduced.

        reduce_func

          Function to reduce partitioned version of intermediate
          data to final output. Takes as argument a key as
          produced by map_func and a sequence of the values
          associated with that key.

        num_workers

          The number of workers to create in the pool. Defaults
          to the number of CPUs available on the current host.
        """
        self.map_func = map_func
        self.reduce_func = reduce_func
        self.pool = multiprocessing.Pool(num_workers)

    def partition(self, mapped_values):
        """Organize the mapped values by their key.
        Returns an unsorted sequence of tuples with a key
        and a sequence of values.
        """
        partitioned_data = collections.defaultdict(list)
        for key, value in mapped_values:
            partitioned_data[key].append(value)
        return partitioned_data.items()

    def __call__(self, inputs, chunksize=1):
        """Process the inputs through the map and reduce functions given.

        inputs
          An iterable containing the input data to be processed.

        chunksize=1
          The portion of the input data to hand to each worker.
          This can be used to tune performance during the mapping phase.
        """
        map_responses = self.pool.map(
            self.map_func,
            inputs,
            chunksize=chunksize,
        )
        partitioned_data = self.partition(
            itertools.chain(*map_responses)
        )
        reduced_values = self.pool.map(
            self.reduce_func,
            partitioned_data,
        )
        return reduced_values
```
下面的示例脚本使用SimpleMapReduce来计算本文的reStructuredText源中的“单词”，忽略了一些标记。

```python
#multiprocessing_wordcount.py
import multiprocessing
import string

from multiprocessing_mapreduce import SimpleMapReduce

def file_to_words(filename):
    """Read a file and return a sequence of
    (word, occurences) values.
    """
    STOP_WORDS = set([
        'a', 'an', 'and', 'are', 'as', 'be', 'by', 'for', 'if',
        'in', 'is', 'it', 'of', 'or', 'py', 'rst', 'that', 'the',
        'to', 'with',
    ])
    TR = str.maketrans({
        p: ' '
        for p in string.punctuation
    })

    print('{} reading {}'.format(multiprocessing.current_process().name, filename))
    output = []

    with open(filename, 'rt') as f:
        for line in f:
            # Skip comment lines.
            if line.lstrip().startswith('..'):
                continue
            line = line.translate(TR)  # Strip punctuation
            for word in line.split():
                word = word.lower()
                if word.isalpha() and word not in STOP_WORDS:
                    output.append((word, 1))
    return output

def count_words(item):
    """Convert the partitioned data for a word to a
    tuple containing the word and the number of occurences.
    """
    word, occurences = item
    return (word, sum(occurences))

if __name__ == '__main__':
    import operator
    import glob

    input_files = glob.glob('*.rst')

    mapper = SimpleMapReduce(file_to_words, count_words)
    word_counts = mapper(input_files)
    word_counts.sort(key=operator.itemgetter(1))
    word_counts.reverse()

    print('\nTOP 20 WORDS BY FREQUENCY\n')
    top20 = word_counts[:20]
    longest = max(len(word) for word, count in top20)
    for word, count in top20:
        print('{word:<{len}}: {count:5}'.format(
            len=longest + 1,
            word=word,
            count=count)
        )
```
`file_to_words（）`函数将每个输入文件转换为包含单词和数字1（表示单次出现）的元组序列。使用单词作为键通过partition（）将数据分割，因此得到的结构由一个键和一个1的序列组成，该序列表示每个单词的出现。在reduction阶段，使用`count_words（）`将分区数据转换为一组元组，其中包含一个单词和对该单词的计数。

```
$ python3 -u multiprocessing_wordcount.py

ForkPoolWorker-1 reading basics.rst
ForkPoolWorker-2 reading communication.rst
ForkPoolWorker-3 reading index.rst
ForkPoolWorker-4 reading mapreduce.rst

TOP 20 WORDS BY FREQUENCY

process         :    83
running         :    45
multiprocessing :    44
worker          :    40
starting        :    37
now             :    35
after           :    34
processes       :    31
start           :    29
header          :    27
pymotw          :    27
caption         :    27
end             :    27
daemon          :    22
can             :    22
exiting         :    21
forkpoolworker  :    21
consumer        :    20
main            :    18
event           :    16
```

## asyncio — Asynchronous I/O, event loop, and concurrency tools

目的：异步I/O和并发框架。  
asyncio模块提供了使用协程构建并发应用程序的工具。虽然threading模块通过应用程序线程实现并发，而multiprocessing使用系统进程实现并发，但asyncio使用单线程，单进程方法，其中应用程序的各个部分相互协作以在最佳时间显式切换任务。大多数情况下，当程序等待读取或写入数据时，会发生此上下文切换，否则阻塞，但asyncio还包括支持“调度代码以在特定的未来时间运行，以使一个协程能够等待另一个协程完成”，支持处理系统信号 ，以及支持识别其他事件，这些事件可能是应用程序改变其工作的原因。

### Asynchronous Concurrency Concepts

大多数使用其他并发模型的程序都是线性编写的，并依赖于语言运行时库或操作系统的底层线程或进程管理来适当地更改上下文。基于asyncio的应用程序要求应用程序代码显式处理上下文更改，并且正确使用这些技术取决于理解几个相互关联的概念。

asyncio提供的框架以事件循环为中心，事件循环是负责高效处理I/O事件、系统事件和应用程序上下文更改的第一类对象。提供了几个循环实现，以有效地利用操作系统功能。虽然通常会自动选择合理的默认值，但也可以从应用程序中选择特定的事件循环实现。这在Windows下很有用，例如，某些循环类以“可能牺牲一些网络I/O处理效率的”方式添加对外部进程的支持。

应用程序显式地与事件循环交互以注册要运行的代码，并允许事件循环在资源可用时对应用程序代码进行必要的调用。例如，网络服务器打开套接字，然后注册它们，以便在输入事件发生时通知它。当有新的传入连接或有数据要读取时，事件循环会通知服务器代码。应用程序代码预期会在短时间内在当前上下文中没有更多的工作可以做时再次让出控制权。例如，如果没有更多数据要从套接字读取，则服务器应该将控制权交还给事件循环。

将控制权交还给事件循环的机制取决于Python的协程，这些特殊函数可以放弃对调用者的控制而不会丢失其状态。协程类似于生成器函数，实际上生成器可用于在早于3.5的Python版本（不支持原生协程对象）中实现协程。 asyncio还为协议（protocols）和传输（transports）提供了一个基于类的抽象层，用于使用回调编写代码代替直接编写协程。在基于类和协程的模型中，通过重新进入事件循环显式更改上下文取代了Python的线程实现中的隐式上下文更改。

future是表示尚未完成的工作结果的数据结构。事件循环可以监视被设置为要完成的Future对象，允许应用程序的一部分等待另一部分完成某些工作。除了futures，asyncio还包括其他并发原语，如锁和信号量。

Task是Future的子类，它知道如何包装和管理协程的执行。可以使用事件循环来调度任务，以便在他们需要的资源可用时运行，并生成可由其他协程使用的结果。

### Cooperative Multitasking with Coroutines

协程是为并发操作而设计的语言结构。协程函数在调用时创建一个协程对象，然后调用者可以使用协程的send（）方法运行该函数的代码。协程可以使用await关键字和另一个协程以暂停执行。当它被暂停时，协程的状态被保持，允许它在下次被唤醒时从它停止的地方恢复。

#### Starting a Coroutine

有几种不同的方法可以让asyncio事件循环启动协程。最简单的方法是使用`run_until_complete（）`，直接将协程传递给它。

```python
#asyncio_coroutine.py
import asyncio

async def coroutine():
    print('in coroutine')

event_loop = asyncio.get_event_loop()
try:
    print('starting coroutine')
    coro = coroutine()
    print('entering event loop')
    event_loop.run_until_complete(coro)
finally:
    print('closing event loop')
    event_loop.close()
```
第一步是获取对事件循环的引用。可以使用默认循环类型，也可以实例化特定的循环类。在此示例中，使用默认循环。`run_until_complete（）`方法使用coroutine对象启动循环，并在协程返回退出时停止循环。

```
$ python3 asyncio_coroutine.py

starting coroutine
entering event loop
in coroutine
closing event loop
```

#### Returning Values from Coroutines

协程的返回值被传递回启动并等待它的代码。

```python
#asyncio_coroutine_return.py
import asyncio

async def coroutine():
    print('in coroutine')
    return 'result'

event_loop = asyncio.get_event_loop()
try:
    return_value = event_loop.run_until_complete(coroutine())
    print('it returned: {!r}'.format(return_value))
finally:
    event_loop.close()
```
在这种情况下，`run_until_complete（）`也返回它正在等待的协程的结果。

```
$ python3 asyncio_coroutine_return.py

in coroutine
it returned: 'result'
```

#### Chaining Coroutines

一个协程可以启动另一个协程并等待结果。这样可以更轻松地将任务分解为可重用的部分。以下示例包含两个必须按顺序执行的阶段，但可以与其他操作同时运行。

```python
#asyncio_coroutine_chain.py
import asyncio

async def outer():
    print('in outer')
    print('waiting for result1')
    result1 = await phase1()
    print('waiting for result2')
    result2 = await phase2(result1)
    return (result1, result2)

async def phase1():
    print('in phase1')
    return 'result1'

async def phase2(arg):
    print('in phase2')
    return 'result2 derived from {}'.format(arg)

event_loop = asyncio.get_event_loop()
try:
    return_value = event_loop.run_until_complete(outer())
    print('return value: {!r}'.format(return_value))
finally:
    event_loop.close()
```
使用await关键字而不是将新的协程添加到循环中，因为控制流已经包含在循环所管理的协程内，因此不必告诉循环来管理新的协程。

```
$ python3 asyncio_coroutine_chain.py

in outer
waiting for result1
in phase1
waiting for result2
in phase2
return value: ('result1', 'result2 derived from result1')
```

#### Generators Instead of Coroutines

协程函数是asyncio设计的关键组成部分。它们提供了一种语言结构，用于停止程序某部分的执行，保留该调用的状态，以及稍后重新进入状态，这些都是并发框架的重要能力。  
Python3.5引入了新的语言功能，使用async def原生地定义这种协程，并使用await让出控制权，asyncio的示例利用了新功能。早期版本的Python3可以使用“用asyncio.coroutine（）装饰器包装的”生成器函数和yield from来实现相同的效果。

```python
#asyncio_generator.py
import asyncio

@asyncio.coroutine
def outer():
    print('in outer')
    print('waiting for result1')
    result1 = yield from phase1()
    print('waiting for result2')
    result2 = yield from phase2(result1)
    return (result1, result2)

@asyncio.coroutine
def phase1():
    print('in phase1')
    return 'result1'

@asyncio.coroutine
def phase2(arg):
    print('in phase2')
    return 'result2 derived from {}'.format(arg)

event_loop = asyncio.get_event_loop()
try:
    return_value = event_loop.run_until_complete(outer())
    print('return value: {!r}'.format(return_value))
finally:
    event_loop.close()
```
前面的示例使用生成器函数而不是原生协程来重现`asyncio_coroutine_chain.py`。

```
$ python3 asyncio_generator.py

in outer
waiting for result1
in phase1
waiting for result2
in phase2
return value: ('result1', 'result2 derived from result1')
```

### Scheduling Calls to Regular Functions

除了管理协程和I/O回调之外，asyncio事件循环还可以根据循环中保留的计时器值来调度对常规函数的调用。

#### Scheduling a Callback “Soon”

如果回调的时间无关紧要，可以使用call_soon（）来调度循环的下一次迭代的调用。函数之后的任何额外位置参数都会传递给回调函数。要将关键字参数传递给回调，请使用functools模块中的partial（）。

```python
#asyncio_call_soon.py
import asyncio
import functools

def callback(arg, *, kwarg='default'):
    print('callback invoked with {} and {}'.format(arg, kwarg))

async def main(loop):
    print('registering callbacks')
    loop.call_soon(callback, 1)
    wrapped = functools.partial(callback, kwarg='not default')
    loop.call_soon(wrapped, 2)
    await asyncio.sleep(0.1)

event_loop = asyncio.get_event_loop()
try:
    print('entering event loop')
    event_loop.run_until_complete(main(event_loop))
finally:
    print('closing event loop')
    event_loop.close()
```
回调按调度顺序调用。

```
$ python3 asyncio_call_soon.py

entering event loop
registering callbacks
callback invoked with 1 and default
callback invoked with 2 and not default
closing event loop
```

#### Scheduling a Callback with a Delay

要将回调推迟到将来某个时间，请使用call_later（）。第一个参数是以秒为单位的延迟，第二个参数是回调。

```python
#asyncio_call_later.py
import asyncio

def callback(n):
    print('callback {} invoked'.format(n))

async def main(loop):
    print('registering callbacks')
    loop.call_later(0.2, callback, 1)
    loop.call_later(0.1, callback, 2)
    loop.call_soon(callback, 3)
    await asyncio.sleep(0.4)

event_loop = asyncio.get_event_loop()
try:
    print('entering event loop')
    event_loop.run_until_complete(main(event_loop))
finally:
    print('closing event loop')
    event_loop.close()
```
在此示例中，相同的函数在不同的时间以不同的参数被调度。最后一个实例，使用call_soon（），导致带有参数3的回调被调用在任何含时间的调度实例之前，说明“soon”通常意味着最小延迟。

```
$ python3 asyncio_call_later.py

entering event loop
registering callbacks
callback 3 invoked
callback 2 invoked
callback 1 invoked
closing event loop
```

#### Scheduling a Callback for a Specific Time

调度调用也可以发生在一个特殊的时间。该循环使用单调时钟，而不是墙上时间，以确保“now”的值永远不会回归（regress）。要选择调度回调的时间，必须使用循环的time（）方法从该时钟的内部状态开始。

```python
#asyncio_call_at.py
import asyncio
import time

def callback(n, loop):
    print('callback {} invoked at {}'.format(n, loop.time()))

async def main(loop):
    now = loop.time()
    print('clock time: {}'.format(time.time()))
    print('loop  time: {}'.format(now))
    print('registering callbacks')
    loop.call_at(now + 0.2, callback, 1, loop)
    loop.call_at(now + 0.1, callback, 2, loop)
    loop.call_soon(callback, 3, loop)
    await asyncio.sleep(1)

event_loop = asyncio.get_event_loop()
try:
    print('entering event loop')
    event_loop.run_until_complete(main(event_loop))
finally:
    print('closing event loop')
    event_loop.close()
```
请注意，根据循环的时间与time.time（）返回的值不匹配。

```
$ python3 asyncio_call_at.py

entering event loop
clock time: 1521404411.833459
loop  time: 715855.398664185
registering callbacks
callback 3 invoked at 715855.398744743
callback 2 invoked at 715855.503897727
callback 1 invoked at 715855.601119414
closing event loop
```

### Producing Results Asynchronously

Future代表尚未完成的工作成果。事件循环可以监视Future对象的状态以指示它已完成，允许应用程序的一部分等待另一部分完成某些工作。

#### Waiting for a Future

一个Future就像一个协程，所以任何用于等待协程的技术也可以用来等待要被标记完成的future。此示例将future传递给事件循环的`run_until_complete（）`方法。

```python
#asyncio_future_event_loop.py
import asyncio

def mark_done(future, result):
    print('setting future result to {!r}'.format(result))
    future.set_result(result)

event_loop = asyncio.get_event_loop()
try:
    all_done = asyncio.Future()

    print('scheduling mark_done')
    event_loop.call_soon(mark_done, all_done, 'the result')

    print('entering event loop')
    result = event_loop.run_until_complete(all_done)
    print('returned result: {!r}'.format(result))
finally:
    print('closing event loop')
    event_loop.close()

print('future result: {!r}'.format(all_done.result()))
```
set_result（）被调用后，Future的状态将更改为完成，Future实例将保存传给该方法的result，以便稍后检索。

```
$ python3 asyncio_future_event_loop.py

scheduling mark_done
entering event loop
setting future result to 'the result'
returned result: 'the result'
closing event loop
future result: 'the result'
```
Future也可以与await关键字一起使用，如本例所示。

```python
#asyncio_future_await.py
import asyncio

def mark_done(future, result):
    print('setting future result to {!r}'.format(result))
    future.set_result(result)

async def main(loop):
    all_done = asyncio.Future()

    print('scheduling mark_done')
    loop.call_soon(mark_done, all_done, 'the result')

    result = await all_done
    print('returned result: {!r}'.format(result))

event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main(event_loop))
finally:
    event_loop.close()
```
Future的结果由await返回，因此通常可以使用相同的代码处理常规协程和Future实例。

```
$ python3 asyncio_future_await.py

scheduling mark_done
setting future result to 'the result'
returned result: 'the result'
```

#### Future Callbacks

除了像协程一样工作之外，Future还可以在完成后调用回调。回调按其注册顺序调用。

```python
#asyncio_future_callback.py
import asyncio
import functools

def callback(future, n):
    print('{}: future done: {}'.format(n, future.result()))

async def register_callbacks(all_done):
    print('registering callbacks on future')
    all_done.add_done_callback(functools.partial(callback, n=1))
    all_done.add_done_callback(functools.partial(callback, n=2))

async def main(all_done):
    await register_callbacks(all_done)
    print('setting result of future')
    all_done.set_result('the result')

event_loop = asyncio.get_event_loop()
try:
    all_done = asyncio.Future()
    event_loop.run_until_complete(main(all_done))
finally:
    event_loop.close()
```
回调应该是一个参数，即Future实例。 要将其他参数传递给回调，请使用functools.partial（）创建包装器。

```
$ python3 asyncio_future_callback.py

registering callbacks on future
setting result of future
1: future done: the result
2: future done: the result
```

### Executing Tasks Concurrently

任务是与事件循环交互的主要方式之一。任务包装协程并在他们完成后跟踪。任务是Future的子类，因此其他协程可以等待它们，每个任务都有一个可以在任务完成后检索的结果。

#### Starting a Task

要启动任务，请使用create_task（）创建Task实例。只要循环正在运行且协程不返回，生成的任务将作为事件循环管理的并发操作的一部分运行。

```python
#asyncio_create_task.py
import asyncio

async def task_func():
    print('in task_func')
    return 'the result'

async def main(loop):
    print('creating task')
    task = loop.create_task(task_func())
    print('waiting for {!r}'.format(task))
    return_value = await task
    print('task completed {!r}'.format(task))
    print('return value: {!r}'.format(return_value))

event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main(event_loop))
finally:
    event_loop.close()
```
此示例等待任务在main（）函数退出之前返回结果。

```
$ python3 asyncio_create_task.py

creating task
waiting for <Task pending coro=<task_func() running at asyncio_create_task.py:12>>
in task_func
task completed <Task finished coro=<task_func() done, defined at asyncio_create_task.py:12> result='the result'>
return value: 'the result'
```

#### Canceling a Task

通过保存从create_task（）返回的Task对象，可以在任务完成之前取消该任务的操作。

```python
#asyncio_cancel_task.py
import asyncio

async def task_func():
    print('in task_func')
    return 'the result'

async def main(loop):
    print('creating task')
    task = loop.create_task(task_func())

    print('canceling task')
    task.cancel()

    print('canceled task {!r}'.format(task))
    try:
        await task
    except asyncio.CancelledError:
        print('caught error from canceled task')
    else:
        print('task result: {!r}'.format(task.result()))

event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main(event_loop))
finally:
    event_loop.close()
```
此示例在启动事件循环之前创建然后取消任务。 结果是`run_until_complete（）`产生CancelledError异常。

```
$ python3 asyncio_cancel_task.py

creating task
canceling task
canceled task <Task cancelling coro=<task_func() running at asyncio_cancel_task.py:12>>
caught error from canceled task
```
如果任务在等待另一个并发操作时被取消，则会通过在等待的位置引发CancelledError异常来通知任务它被取消。

```python
#asyncio_cancel_task2.py
import asyncio

async def task_func():
    print('in task_func, sleeping')
    try:
        await asyncio.sleep(1)
    except asyncio.CancelledError:
        print('task_func was canceled')
        raise
    return 'the result'

def task_canceller(t):
    print('in task_canceller')
    t.cancel()
    print('canceled the task')

async def main(loop):
    print('creating task')
    task = loop.create_task(task_func())
    loop.call_soon(task_canceller, task)
    try:
        await task
    except asyncio.CancelledError:
        print('main() also sees task as canceled')

event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main(event_loop))
finally:
    event_loop.close()
```
如有必要，捕获异常提供了清理已完成工作的机会。

```
$ python3 asyncio_cancel_task2.py

creating task
in task_func, sleeping
in task_canceller
canceled the task
task_func was canceled
main() also sees task as canceled
```

#### Creating Tasks from Coroutines

ensure_future（）函数返回一个Task绑定到一个协程的执行。然后可以将该Task实例传递给其他代码，这些代码可以在不知道原始协程如何被构造或调用的情况下等待它。

```python
#asyncio_ensure_future.py
import asyncio

async def wrapped():
    print('wrapped')
    return 'result'

async def inner(task):
    print('inner: starting')
    print('inner: waiting for {!r}'.format(task))
    result = await task
    print('inner: task returned {!r}'.format(result))

async def starter():
    print('starter: creating task')
    task = asyncio.ensure_future(wrapped())
    print('starter: waiting for inner')
    await inner(task)
    print('starter: inner returned')

event_loop = asyncio.get_event_loop()
try:
    print('entering event loop')
    result = event_loop.run_until_complete(starter())
finally:
    event_loop.close()
```
请注意，传递给ensure_future（）的协程不会启动，直到使用await来允许执行它。

```
$ python3 asyncio_ensure_future.py

entering event loop
starter: creating task
starter: waiting for inner
inner: starting
inner: waiting for <Task pending coro=<wrapped() running at asyncio_ensure_future.py:12>>
wrapped
inner: task returned 'result'
starter: inner returned
```

### Composing Coroutines with Control Structures

使用内置语言关键字await，使一系列协程之间的线性控制流易于管理。使用asyncio中的工具也可以实现更复杂的结构，允许一个协程等待其他几个协程并行完成。

#### Waiting for Multiple Coroutines

将一个操作分成许多部分并分别执行它们通常很有用。例如，下载多个远程资源或查询远程API。在执行顺序无关紧要的情况下，以及可能存在任意数量的操作的情况下，wait（）可用于暂停一个协程，直到其他后台操作完成。

```python
#asyncio_wait.py
import asyncio

async def phase(i):
    print('in phase {}'.format(i))
    await asyncio.sleep(0.1 * i)
    print('done with phase {}'.format(i))
    return 'phase {} result'.format(i)

async def main(num_phases):
    print('starting main')
    phases = [ phase(i) for i in range(num_phases)]
    print('waiting for phases to complete')
    completed, pending = await asyncio.wait(phases)
    results = [t.result() for t in completed]
    print('results: {!r}'.format(results))

event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main(3))
finally:
    event_loop.close()
```
在内部，wait（）使用一个set来保存它创建的Task实例。这导致它们以不可预测的顺序开始和结束。wait（）的返回值是一个元组，包含两个“包含完成和挂起任务的”集合（set）。

```
$ python3 asyncio_wait.py

starting main
waiting for phases to complete
in phase 0
in phase 1
in phase 2
done with phase 0
done with phase 1
done with phase 2
results: ['phase 1 result', 'phase 0 result', 'phase 2 result']
```
只有wait（）与超时值一起使用时，才会有挂起的操作返回。

```python
#asyncio_wait_timeout.py
import asyncio

async def phase(i):
    print('in phase {}'.format(i))
    try:
        await asyncio.sleep(0.1 * i)
    except asyncio.CancelledError:
        print('phase {} canceled'.format(i))
        raise
    else:
        print('done with phase {}'.format(i))
        return 'phase {} result'.format(i)

async def main(num_phases):
    print('starting main')
    phases = [ phase(i) for i in range(num_phases) ]
    print('waiting 0.1 for phases to complete')
    completed, pending = await asyncio.wait(phases, timeout=0.1)
    print('{} completed and {} pending'.format(
        len(completed), len(pending),
    ))
    # Cancel remaining tasks so they do not generate errors
    # as we exit without finishing them.
    if pending:
        print('canceling tasks')
        for t in pending:
            t.cancel()
    print('exiting main')

event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main(3))
finally:
    event_loop.close()
```
那些剩余的后台操作应该被等待他们的代码（函数）取消或结束。在事件循环继续时，让它们保持挂起将使它们进一步执行，如果整个操作被被认为已中止，这可能是不可取的。在进程结束时仍将其保持挂起将导致报告一个警告。

```
$ python3 asyncio_wait_timeout.py

starting main
waiting 0.1 for phases to complete
in phase 1
in phase 0
in phase 2
done with phase 0
1 completed and 2 pending
cancelling tasks
exiting main
phase 1 cancelled
phase 2 cancelled
```

#### Gathering Results from Coroutines

如果后台阶段（phase）定义明确，并且只有这些阶段的结果很重要，那么gather（）对于等待多个操作可能更有用。

```python
#asyncio_gather.py
import asyncio

async def phase1():
    print('in phase1')
    await asyncio.sleep(2)
    print('done with phase1')
    return 'phase1 result'

async def phase2():
    print('in phase2')
    await asyncio.sleep(1)
    print('done with phase2')
    return 'phase2 result'

async def main():
    print('starting main')
    print('waiting for phases to complete')
    results = await asyncio.gather(
        phase1(),
        phase2(),
    )
    print('results: {!r}'.format(results))

event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main())
finally:
    event_loop.close()
```
gather创建的任务不会公开，因此无法取消。返回值是一个结果列表，其顺序与传递给gather（）的参数的顺序相同，与后台操作实际完成的顺序无关。

```
$ python3 asyncio_gather.py

starting main
waiting for phases to complete
in phase2
in phase1
done with phase2
done with phase1
results: ['phase1 result', 'phase2 result']
```

#### Handling Background Operations as They Finish

`as_completed（）`是一个生成器，它管理传给它的协程列表的执行，并在它们结束运行时一次生成一个结果。与`wait（）`一样，`as_completed（）`不保证顺序，但在采取其他操作之前不必等待所有后台操作完成。

```python
#asyncio_as_completed.py
import asyncio

async def phase(i):
    print('in phase {}'.format(i))
    await asyncio.sleep(0.5 - (0.1 * i))
    print('done with phase {}'.format(i))
    return 'phase {} result'.format(i)

async def main(num_phases):
    print('starting main')
    phases = [phase(i) for i in range(num_phases)]
    print('waiting for phases to complete')
    results = []
    for next_to_complete in asyncio.as_completed(phases):
        answer = await next_to_complete
        print('received answer {!r}'.format(answer))
        results.append(answer)
    print('results: {!r}'.format(results))
    return results

event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main(3))
finally:
    event_loop.close()
```
此示例启动了几个后台阶段，这些阶段以相反的顺序完成。当生成器被消耗时，循环使用await等待协程的结果。

```
$ python3 asyncio_as_completed.py

starting main
waiting for phases to complete
in phase 0
in phase 2
in phase 1
done with phase 2
received answer 'phase 2 result'
done with phase 1
received answer 'phase 1 result'
done with phase 0
received answer 'phase 0 result'
results: ['phase 2 result', 'phase 1 result', 'phase 0 result']
```

### Synchronization Primitives

虽然asyncio应用程序通常作为单线程进程运行，但它们仍然构建为并发应用程序。基于来自I/O和其他外部事件的延迟和中断，每个协程或任务以不可预测的顺序执行。为了支持安全并发，asyncio包括“在threading和multiprocessing模块中找到的一些相同的”低级原语的实现。

#### Locks

Lock可用于保护对共享资源的访问。只有锁的持有者才能使用该资源。多次尝试获取锁将阻塞，以便一次只有一个持有者。

```python
#asyncio_lock.py
import asyncio
import functools

def unlock(lock):
    print('callback releasing lock')
    lock.release()

async def coro1(lock):
    print('coro1 waiting for the lock')
    with await lock:
        print('coro1 acquired lock')
    print('coro1 released lock')

async def coro2(lock):
    print('coro2 waiting for the lock')
    await lock
    try:
        print('coro2 acquired lock')
    finally:
        print('coro2 released lock')
        lock.release()

async def main(loop):
    # Create and acquire a shared lock.
    lock = asyncio.Lock()
    print('acquiring the lock before starting coroutines')
    await lock.acquire()
    print('lock acquired: {}'.format(lock.locked()))

    # Schedule a callback to unlock the lock.
    loop.call_later(0.1, functools.partial(unlock, lock))

    # Run the coroutines that want to use the lock.
    print('waiting for coroutines')
    await asyncio.wait([coro1(lock), coro2(lock)]),

event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main(event_loop))
finally:
    event_loop.close()
```
可以直接调用锁，使用await获取它并在完成时调用release（）方法，如本示例中的coro2（）。它们也可以“使用with wait 关键字”作为异步上下文管理器使用，如coro1（）。

```
$ python3 asyncio_lock.py

acquiring the lock before starting coroutines
lock acquired: True
waiting for coroutines
coro1 waiting for the lock
coro2 waiting for the lock
callback releasing lock
coro1 acquired lock
coro1 released lock
coro2 acquired lock
coro2 released lock
```

#### Events

asyncio.Event基于threading.Event，用于允许多个使用者等待某些事情发生，而无需查找“要被关联到通知的”特定值。

```python
#asyncio_event.py
import asyncio
import functools

def set_event(event):
    print('setting event in callback')
    event.set()

async def coro1(event):
    print('coro1 waiting for event')
    await event.wait()
    print('coro1 triggered')

async def coro2(event):
    print('coro2 waiting for event')
    await event.wait()
    print('coro2 triggered')

async def main(loop):
    # Create a shared event
    event = asyncio.Event()
    print('event start state: {}'.format(event.is_set()))

    loop.call_later( 0.1, functools.partial(set_event, event) )

    await asyncio.wait([coro1(event), coro2(event)])
    print('event end state: {}'.format(event.is_set()))

event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main(event_loop))
finally:
    event_loop.close()
```
与Lock一样，coro1（）和coro2（）都等待事件被设置。不同之处在于，一旦事件状态发生变化，两者都可以启动，并且它们不需要获取事件对象的唯一持有权。

```
$ python3 asyncio_event.py

event start state: False
coro2 waiting for event
coro1 waiting for event
setting event in callback
coro2 triggered
coro1 triggered
event end state: True
```

#### Conditions

Condition与Event的工作方式类似，除了不是通知所有等待的协程，而是由notify（）参数控制唤醒的等待者数量。

```python
#asyncio_condition.py
import asyncio

async def consumer(condition, n):
    with await condition:
        print('consumer {} is waiting'.format(n))
        await condition.wait()
        print('consumer {} triggered'.format(n))
    print('ending consumer {}'.format(n))

async def manipulate_condition(condition):
    print('starting manipulate_condition')

    # pause to let consumers start
    await asyncio.sleep(0.1)

    for i in range(1, 3):
        with await condition:
            print('notifying {} consumers'.format(i))
            condition.notify(n=i)
        await asyncio.sleep(0.1)

    with await condition:
        print('notifying remaining consumers')
        condition.notify_all()

    print('ending manipulate_condition')

async def main(loop):
    # Create a condition
    condition = asyncio.Condition()

    # Set up tasks watching the condition
    consumers = [ consumer(condition, i) for i in range(5) ]

    # Schedule a task to manipulate the condition variable
    loop.create_task(manipulate_condition(condition))

    # Wait for the consumers to be done
    await asyncio.wait(consumers)

event_loop = asyncio.get_event_loop()
try:
    result = event_loop.run_until_complete(main(event_loop))
finally:
    event_loop.close()
```
此示例启动Condition的五个消费者。每一个都使用wait（）方法等待可以使他们继续的通知。`manipulate_condition（）`通知一个消费者，然后是两个消费者，然后是所有剩余的消费者。

```
$ python3 asyncio_condition.py

starting manipulate_condition
consumer 3 is waiting
consumer 1 is waiting
consumer 2 is waiting
consumer 0 is waiting
consumer 4 is waiting
notifying 1 consumers
consumer 3 triggered
ending consumer 3
notifying 2 consumers
consumer 1 triggered
ending consumer 1
consumer 2 triggered
ending consumer 2
notifying remaining consumers
ending manipulate_condition
consumer 0 triggered
ending consumer 0
consumer 4 triggered
ending consumer 4
```

#### Queues

asyncio.Queue为协程提供了先进先出的数据结构，就像queue.Queue用于线程或multiprocessing.Queue用于进程。

```python
#asyncio_queue.py
import asyncio

async def consumer(n, q):
    print('consumer {}: starting'.format(n))
    while True:
        print('consumer {}: waiting for item'.format(n))
        item = await q.get()
        print('consumer {}: has item {}'.format(n, item))
        if item is None:
            # None is the signal to stop.
            q.task_done()
            break
        else:
            await asyncio.sleep(0.01 * item)
            q.task_done()
    print('consumer {}: ending'.format(n))

async def producer(q, num_workers):
    print('producer: starting')
    # Add some numbers to the queue to simulate jobs
    for i in range(num_workers * 3):
        await q.put(i)
        print('producer: added task {} to the queue'.format(i))
    # Add None entries in the queue
    # to signal the consumers to exit
    print('producer: adding stop signals to the queue')
    for i in range(num_workers):
        await q.put(None)
    print('producer: waiting for queue to empty')
    await q.join()
    print('producer: ending')

async def main(loop, num_consumers):
    # Create the queue with a fixed size so the producer
    # will block until the consumers pull some items out.
    q = asyncio.Queue(maxsize=num_consumers)

    # Scheduled the consumer tasks.
    consumers = [
        loop.create_task(consumer(i, q))
        for i in range(num_consumers)
    ]

    # Schedule the producer task.
    prod = loop.create_task(producer(q, num_consumers))

    # Wait for all of the coroutines to finish.
    await asyncio.wait(consumers + [prod])

event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main(event_loop, 2))
finally:
    event_loop.close()
```
使用put（）添加项目或使用get（）删除项目都是异步操作，因为队列大小可能是固定的（阻止添加）或队列可能为空（阻止调用获取项目）。

```python
$ python3 asyncio_queue.py

consumer 0: starting
consumer 0: waiting for item
consumer 1: starting
consumer 1: waiting for item
producer: starting
producer: added task 0 to the queue
producer: added task 1 to the queue
consumer 0: has item 0
consumer 1: has item 1
producer: added task 2 to the queue
producer: added task 3 to the queue
consumer 0: waiting for item
consumer 0: has item 2
producer: added task 4 to the queue
consumer 1: waiting for item
consumer 1: has item 3
producer: added task 5 to the queue
producer: adding stop signals to the queue
consumer 0: waiting for item
consumer 0: has item 4
consumer 1: waiting for item
consumer 1: has item 5
producer: waiting for queue to empty
consumer 0: waiting for item
consumer 0: has item None
consumer 0: ending
consumer 1: waiting for item
consumer 1: has item None
consumer 1: ending
producer: ending
```

### Asynchronous I/O with Protocol Class Abstractions

到目前为止，这些示例都避免了混合并发和I/O操作，以便一次关注一个概念。然而，当I/O阻塞时切换上下文，是asyncio的主要用例之一。在已经介绍的并发概念的基础上，本节将介绍两个示例程序，实现简单回显服务器和客户端，类似于socket和socketserver部分中使用的示例。客户端可以连接到服务器，发送一些数据，然后接收与响应相同的数据。每次启动I/O操作时，执行代码都会放弃对事件循环的控制，允许其他任务运行直到I/O准备就绪。

#### Echo Server

服务器首先导入它需要设置的模块asyncio和logging，然后创建一个事件循环对象。

```python
#asyncio_echo_server_protocol.py
import asyncio
import logging
import sys

SERVER_ADDRESS = ('localhost', 10000)

logging.basicConfig(
    level=logging.DEBUG,
    format='%(name)s: %(message)s',
    stream=sys.stderr,
)
log = logging.getLogger('main')

event_loop = asyncio.get_event_loop()
```
然后，它定义了asyncio.Protocol的一个子类来处理客户端通信。根据与服务器套接字关联的事件调用协议对象的方法。

```python
class EchoServer（asyncio.Protocol）：
```
每个新客户端连接都会触发对`connection_made（）`的调用。
transport参数是asyncio.Transport的一个实例，它提供了使用套接字进行异步I/O的抽象。
不同类型的通信提供不同的传输实现，但具有相同的API。例如，有单独的传输类用于处理套接字和处理到子进程的管道。
传入客户端的地址可通过`get_extra_info（）`从传输获得，它是一种特定于实现的方法。

```python
    def connection_made(self, transport):
        self.transport = transport
        self.address = transport.get_extra_info('peername')
        self.log = logging.getLogger('EchoServer_{}_{}'.format(*self.address) )
        self.log.debug('connection accepted')
```
建立连接后，当数据从客户端发送到服务器时，将调用协议的data_received（）方法以传入数据进行处理。数据作为字节字符串传递，由应用程序以适当的方式对其进行解码。这里记录结果，然后通过调用transport.write（）立即将响应发送回客户端。

```python
    def data_received(self, data):
        self.log.debug('received {!r}'.format(data))
        self.transport.write(data)
        self.log.debug('sent {!r}'.format(data))
```
某些传输支持特殊的文件结束指示符（“EOF”）。遇到EOF时，将调用eof_received（）方法。在此实现中，EOF被发送回客户端以指示它已被接收。由于并非所有传输都支持显式EOF，因此该协议首先询问传输是否可以安全地发送EOF。

```python
    def eof_received(self):
        self.log.debug('received EOF')
        if self.transport.can_write_eof():
            self.transport.write_eof()
```
当连接正常关闭或由于错误而关闭时，将调用协议的connection_lost（）方法。如果存在错误，则参数包含适当的异常对象。否则它是None。

```python
    def connection_lost(self, error):
        if error:
            self.log.error('ERROR: {}'.format(error))
        else:
            self.log.debug('closing')
        super().connection_lost(error)
```
启动服务器有两个步骤。首先，应用程序告诉事件循环“使用协议类以及主机名和要监听的端口”创建新的服务器对象。create_server（）方法是一个协程，因此必须由事件循环处理它的结果才能实际启动服务器。完成协程会生成绑定到该事件循环的asyncio.Server实例。

```python
# Create the server and let the loop finish the coroutine before
# starting the real event loop.
factory = event_loop.create_server(EchoServer, *SERVER_ADDRESS)
server = event_loop.run_until_complete(factory)
log.debug('starting up on {} port {}'.format(*SERVER_ADDRESS))
```
然后需要运行事件循环以处理事件和处理客户端请求。对于长时间运行的服务，run_forever（）方法是执行此操作的最简单方法。当事件循环停止时，无论是通过应用程序代码还是通过发出进程信号，可以关闭服务器以正确清理套接字，然后可以关闭事件循环以在程序退出之前完成处理任何其他协程。

```python
# Enter the event loop permanently to handle all connections.
try:
    event_loop.run_forever()
finally:
    log.debug('closing server')
    server.close()
    event_loop.run_until_complete(server.wait_closed())
    log.debug('closing event loop')
    event_loop.close()
```

#### Echo Client

使用协议类构造客户端与构建服务器非常相似。代码再次首先导入需要设置的模块asyncio和logging，然后创建一个事件循环对象。

```python
#asyncio_echo_client_protocol.py
import asyncio
import functools
import logging
import sys

MESSAGES = [
    b'This is the message. ',
    b'It will be sent ',
    b'in parts.',
]
SERVER_ADDRESS = ('localhost', 10000)

logging.basicConfig(
    level=logging.DEBUG,
    format='%(name)s: %(message)s',
    stream=sys.stderr,
)
log = logging.getLogger('main')

event_loop = asyncio.get_event_loop()
```
客户端协议类定义的方法与服务器相同，但具有不同的实现。类构造函数接受两个参数，一个要发送的消息列表和一个Future实例，Future用于通过接收来自服务器的响应以通知客户端已完成一个工作周期。

```python
class EchoClient(asyncio.Protocol):

    def __init__(self, messages, future):
        super().__init__()
        self.messages = messages
        self.log = logging.getLogger('EchoClient')
        self.f = future
```
当客户端成功连接到服务器时，它立即开始通信。消息序列被一次一个地发送，尽管底层网络代码可以将多个消息组合成一个传输。当所有消息都耗尽时，将发送EOF。  
虽然看起来数据都是立即发送的，但实际上传输对象会缓冲传出的数据并设置回调，以便在套接字缓冲区准备好接收数据时真正的传输数据。这一切都是透明处理的，因此编写应用程序代码可以假设I/O操作马上发生。

```python
    def connection_made(self, transport):
        self.transport = transport
        self.address = transport.get_extra_info('peername')
        self.log.debug('connecting to {} port {}'.format(*self.address) )
        # This could be transport.writelines() except that
        # would make it harder to show each part of the message
        # being sent.
        for msg in self.messages:
            transport.write(msg)
            self.log.debug('sending {!r}'.format(msg))
        if transport.can_write_eof():
            transport.write_eof()
```
收到服务器的响应后，将记录该响应。

```python
    def data_received(self, data):
        self.log.debug('received {!r}'.format(data))
```
当接收到文件结束标记或由服务器端关闭连接时，本地传输对象将被关闭，并通过设置result将future对象标记为已完成。

```python
    def eof_received(self):
        self.log.debug('received EOF')
        self.transport.close()
        if not self.f.done():
            self.f.set_result(True)

    def connection_lost(self, exc):
        self.log.debug('server closed connection')
        self.transport.close()
        if not self.f.done():
            self.f.set_result(True)
        super().connection_lost(exc)
```
通常，协议类将传递给事件循环以创建连接。在这种情况下，因为事件循环没有办法将额外参数传递给协议构造函数，所以有必要创建一个partial来包装客户端类并传递要发送的消息列表和Future实例。然后在调用create_connection（）以建立客户端连接时，使用新的callable代替该类。

```python
client_completed = asyncio.Future()

client_factory = functools.partial(
    EchoClient,
    messages=MESSAGES,
    future=client_completed,
)
factory_coroutine = event_loop.create_connection(
    client_factory,
    *SERVER_ADDRESS,
)
```
要触发客户端运行，使用协程调用事件循环一次以创建客户端，然后再次使用传递给客户端的Future实例调用事件循环一次以在完成时进行通信。使用这样的两个调用可以避免在客户端程序中出现无限循环，它可能想要在完成与服务器的通信后退出。如果仅使用第一个调用来等待协程创建客户端，则它可能不会处理所有响应数据并正确清理与服务器的连接。

```python
log.debug('waiting for client to complete')
try:
    event_loop.run_until_complete(factory_coroutine)
    event_loop.run_until_complete(client_completed)
finally:
    log.debug('closing event loop')
    event_loop.close()
```

#### Output

在一个窗口中运行服务器而另一个窗口中运行客户端生成以下输出。

```
$ python3 asyncio_echo_client_protocol.py
asyncio: Using selector: KqueueSelector
main: waiting for client to complete
EchoClient: connecting to ::1 port 10000
EchoClient: sending b'This is the message. '
EchoClient: sending b'It will be sent '
EchoClient: sending b'in parts.'
EchoClient: received b'This is the message. It will be sent in parts.'
EchoClient: received EOF
EchoClient: server closed connection
main: closing event loop

$ python3 asyncio_echo_client_protocol.py
asyncio: Using selector: KqueueSelector
main: waiting for client to complete
EchoClient: connecting to ::1 port 10000
EchoClient: sending b'This is the message. '
EchoClient: sending b'It will be sent '
EchoClient: sending b'in parts.'
EchoClient: received b'This is the message. It will be sent in parts.'
EchoClient: received EOF
EchoClient: server closed connection
main: closing event loop

$ python3 asyncio_echo_client_protocol.py
asyncio: Using selector: KqueueSelector
main: waiting for client to complete
EchoClient: connecting to ::1 port 10000
EchoClient: sending b'This is the message. '
EchoClient: sending b'It will be sent '
EchoClient: sending b'in parts.'
EchoClient: received b'This is the message. It will be sent in parts.'
EchoClient: received EOF
EchoClient: server closed connection
main: closing event loop
```
虽然客户端总是分开发送消息，但客户端第一次运行时服务器会收到一条大消息并将其回送给客户端。这些结果在后续运行中会有所不同，具体取决于网络的繁忙程度以及在准备好所有数据之前是否刷新网络缓冲区。

```
$ python3 asyncio_echo_server_protocol.py
asyncio: Using selector: KqueueSelector
main: starting up on localhost port 10000
EchoServer_::1_63347: connection accepted
EchoServer_::1_63347: received b'This is the message. It will be sent in parts.'
EchoServer_::1_63347: sent b'This is the message. It will be sent in parts.'
EchoServer_::1_63347: received EOF
EchoServer_::1_63347: closing

EchoServer_::1_63387: connection accepted
EchoServer_::1_63387: received b'This is the message. '
EchoServer_::1_63387: sent b'This is the message. '
EchoServer_::1_63387: received b'It will be sent in parts.'
EchoServer_::1_63387: sent b'It will be sent in parts.'
EchoServer_::1_63387: received EOF
EchoServer_::1_63387: closing

EchoServer_::1_63389: connection accepted
EchoServer_::1_63389: received b'This is the message. It will be sent '
EchoServer_::1_63389: sent b'This is the message. It will be sent '
EchoServer_::1_63389: received b'in parts.'
EchoServer_::1_63389: sent b'in parts.'
EchoServer_::1_63389: received EOF
EchoServer_::1_63389: closing
```

### Asynchronous I/O Using Coroutines and Streams

本节将介绍另一个版本的“实现了回显服务器和客户端的”两个样例程序，它使用协程和asyncio流API而不是协议和传输类抽象。这些示例的运行抽象级别低于前面讨论的协议API，但处理的事件是类似的。

#### Echo Server

服务器首先导入需要设置的模块asyncio和logging，然后创建一个事件循环对象。

```python
#asyncio_echo_server_coroutine.py
import asyncio
import logging
import sys

SERVER_ADDRESS = ('localhost', 10000)
logging.basicConfig(
    level=logging.DEBUG,
    format='%(name)s: %(message)s',
    stream=sys.stderr,
)
log = logging.getLogger('main')

event_loop = asyncio.get_event_loop()
```
然后它定义了一个协程来处理通信。每次客户端连接时，都会调用该协程的一个新实例，以便在函数内代码一次只与一个客户端通信。Python的语言运行时管理每个协程实例的状态，因此应用程序代码不需要管理任何额外的数据结构来跟踪每个客户端。  
协程的参数是与新连接关联的StreamReader和StreamWriter实例。与Transport一样，可以通过writer的方法`get_extra_info（）`访问客户端地址。

```python
async def echo(reader, writer):
    address = writer.get_extra_info('peername')
    log = logging.getLogger('echo_{}_{}'.format(*address))
    log.debug('connection accepted')
```
虽然在连接被建立时协程被调用，但可能还没有任何数据可供读取。为了避免在读取时阻塞，协程使用await调用read（）来允许事件循环继续处理其他任务，直到有数据要读取。

```python
    while True:
        data = await reader.read(128)
```
如果客户端发送数据，它将从await返回，并可以通过将其传递给writer发送回客户端。对write（）的多次调用可用于缓冲传出数据，然后使用drain（）来flush缓冲区。由于flush网络I/O可能阻塞，因此再次await用于将控制权传回事件循环，该循环监视写入套接字并在可以发送更多数据时调用writer。

```python
        if data:
            log.debug('received {!r}'.format(data))
            writer.write(data)
            await writer.drain()
            log.debug('sent {!r}'.format(data))
```
如果客户端没有发送任何数据，则read（）返回一个空字节字符串以指示连接已关闭。服务器需要关闭套接字以写入客户端，然后协程可以返回以指示它已完成。

```python
        else:
            log.debug('closing')
            writer.close()
            return
```
启动服务器有两个步骤。首先，应用程序告诉事件循环使用协同程序以及主机名和要监听的端口来创建新的服务器对象。start_server（）方法本身就是一个协程，因此必须由事件循环处理结果才能实际启动服务器。完成协程会生成绑定到事件循环的asyncio.Server实例。

```python
# Create the server and let the loop finish the coroutine before
# starting the real event loop.
factory = asyncio.start_server(echo, *SERVER_ADDRESS)
server = event_loop.run_until_complete(factory)
log.debug('starting up on {} port {}'.format(*SERVER_ADDRESS))
```
然后需要运行事件循环以处理事件和处理客户端请求。对于长时间运行的服务，run_forever（）方法是执行此操作的最简单方法。当事件循环通过应用程序代码或通过发出进程信号停止，可以关闭服务器以正确清理套接字，然后可以关闭事件循环以在程序退出之前完成处理任何其他协程。

```python
# Enter the event loop permanently to handle all connections.
try:
    event_loop.run_forever()
except KeyboardInterrupt:
    pass
finally:
    log.debug('closing server')
    server.close()
    event_loop.run_until_complete(server.wait_closed())
    log.debug('closing event loop')
    event_loop.close()
```

#### Echo Client

使用协程构建客户端与构建服务器非常相似。代码再次首先导入需要设置的模块asyncio和logging，然后创建事件循环对象。

```python
#asyncio_echo_client_coroutine.py
import asyncio
import logging
import sys

MESSAGES = [
    b'This is the message. ',
    b'It will be sent ',
    b'in parts.',
]
SERVER_ADDRESS = ('localhost', 10000)

logging.basicConfig(
    level=logging.DEBUG,
    format='%(name)s: %(message)s',
    stream=sys.stderr,
)
log = logging.getLogger('main')

event_loop = asyncio.get_event_loop()
```
echo_client协程接受参数，告诉它服务器在哪里以及要发送什么消息。

```python
async def echo_client(address, messages):
```
任务启动时会调用该协程，但它没有可用的活动连接。因此，第一步是让客户建立自己的连接。它使用await来避免在open_connection（）协程运行时阻塞其他活动。

```python
    log = logging.getLogger('echo_client')

    log.debug('connecting to {} port {}'.format(*address))
    reader, writer = await asyncio.open_connection(*address)
```
open_connection（）协程返回与新套接字关联的StreamReader和StreamWriter实例。下一步是使用writer将数据发送到服务器。与在服务器中一样，writer将缓冲传出数据，直到套接字准备就绪或使用drain（）来刷新缓冲区。由于刷新网络I/O可能阻塞，因此再次await用于将控制权传回事件循环，该循环监视写入套接字并在可以发送更多数据时调用写入器。

```python
    # This could be writer.writelines() except that
    # would make it harder to show each part of the message
    # being sent.
    for msg in messages:
        writer.write(msg)
        log.debug('sending {!r}'.format(msg))
    if writer.can_write_eof():
        writer.write_eof()
    await writer.drain()
```
接下来，客户端通过尝试读取数据来查找服务器的响应，直到没有任何内容可供读取。为了避免阻塞单个read（）调用，await将控制权返回给事件循环。如果服务器已发送数据，则会记录该数据。如果服务器未发送任何数据，则read（）返回空字节字符串以指示连接已关闭。客户端需要关闭套接字以发送到服务器，然后返回以指示它已完成。

```python
    log.debug('waiting for response')
    while True:
        data = await reader.read(128)
        if data:
            log.debug('received {!r}'.format(data))
        else:
            log.debug('closing')
            writer.close()
            return
```
要启动客户端，将使用协程调用事件循环来创建客户端。使用`run_until_complete（）`可以避免在客户端程序中出现无限循环。与协议示例不同，不需要单独的future来通知协程何时完成，因为echo_client（）本身包含所有客户端逻辑，并且在收到响应并关闭服务器连接之前它不会返回。

```python
try:
    event_loop.run_until_complete(echo_client(SERVER_ADDRESS, MESSAGES) )
finally:
    log.debug('closing event loop')
    event_loop.close()
```

#### Output

在一个窗口中运行服务器而另一个窗口中运行客户端生成以下输出。

```
$ python3 asyncio_echo_client_coroutine.py
asyncio: Using selector: KqueueSelector
echo_client: connecting to localhost port 10000
echo_client: sending b'This is the message. '
echo_client: sending b'It will be sent '
echo_client: sending b'in parts.'
echo_client: waiting for response
echo_client: received b'This is the message. It will be sent in parts.'
echo_client: closing
main: closing event loop

$ python3 asyncio_echo_client_coroutine.py
asyncio: Using selector: KqueueSelector
echo_client: connecting to localhost port 10000
echo_client: sending b'This is the message. '
echo_client: sending b'It will be sent '
echo_client: sending b'in parts.'
echo_client: waiting for response
echo_client: received b'This is the message. It will be sent in parts.'
echo_client: closing
main: closing event loop

$ python3 asyncio_echo_client_coroutine.py
asyncio: Using selector: KqueueSelector
echo_client: connecting to localhost port 10000
echo_client: sending b'This is the message. '
echo_client: sending b'It will be sent '
echo_client: sending b'in parts.'
echo_client: waiting for response
echo_client: received b'This is the message. It will be sent '
echo_client: received b'in parts.'
echo_client: closing
main: closing event loop
```
虽然客户端总是单独发送消息，但客户端的前两次运行服务器接收一条大消息并将其回送给客户端。这些结果在后续运行中会有所不同，具体取决于网络的繁忙程度以及在准备好所有数据之前是否刷新网络缓冲区。

```
$ python3 asyncio_echo_server_coroutine.py
asyncio: Using selector: KqueueSelector
main: starting up on localhost port 10000
echo_::1_64624: connection accepted
echo_::1_64624: received b'This is the message. It will be sent in parts.'
echo_::1_64624: sent b'This is the message. It will be sent in parts.'
echo_::1_64624: closing

echo_::1_64626: connection accepted
echo_::1_64626: received b'This is the message. It will be sent in parts.'
echo_::1_64626: sent b'This is the message. It will be sent in parts.'
echo_::1_64626: closing

echo_::1_64627: connection accepted
echo_::1_64627: received b'This is the message. It will be sent '
echo_::1_64627: sent b'This is the message. It will be sent '
echo_::1_64627: received b'in parts.'
echo_::1_64627: sent b'in parts.'
echo_::1_64627: closing
```

### Using SSL

asyncio内置支持在套接字上启用SSL通信。将SSLContext实例传递给创建服务器或客户端连接的协程会启用该支持，并确保在套接字准备就绪以让应用程序使用之前处理好SSL协议设置。  
上一部分中基于协程的回显服务器和客户端可以使用一些小的更改来更新。第一步是创建证书和密钥文件。可以使用以下命令创建自签名证书：

```
$ openssl req -newkey rsa:2048 -nodes -keyout pymotw.key -x509 -days 365 -out pymotw.crt
```
openssl命令将提示输入用于生成证书的多个值，然后生成所请求的输出文件。  
上一个服务器示例中的不安全套接字设置使用start_server（）来创建侦听套接字。

```python
factory = asyncio.start_server(echo, *SERVER_ADDRESS)
server = event_loop.run_until_complete(factory)
```
要添加加密，请使用刚生成的证书和密钥创建SSLContext，然后将上下文传递给start_server（）。

```python
# The certificate is created with pymotw.com as the hostname,
# which will not match when the example code runs elsewhere,
# so disable hostname verification.
ssl_context = ssl.create_default_context(ssl.Purpose.CLIENT_AUTH)
ssl_context.check_hostname = False
ssl_context.load_cert_chain('pymotw.crt', 'pymotw.key')

# Create the server and let the loop finish the coroutine before
# starting the real event loop.
factory = asyncio.start_server(echo, *SERVER_ADDRESS, ssl=ssl_context)
```
客户端需要进行类似的更改。旧版本使用open_connection（）创建连接到服务器的套接字。

```python
    reader, writer = await asyncio.open_connection(*address)
```
再次需要SSLContext来保护客户端的套接字。客户端身份不是强制的，因此只需加载证书。

```python
    # The certificate is created with pymotw.com as the hostname,
    # which will not match when the example code runs
    # elsewhere, so disable hostname verification.
    ssl_context = ssl.create_default_context(ssl.Purpose.SERVER_AUTH, )
    ssl_context.check_hostname = False
    ssl_context.load_verify_locations('pymotw.crt')
    reader, writer = await asyncio.open_connection(*server_address, ssl=ssl_context)
```
客户端需要另外一个小的改动。由于SSL连接不支持发送文件结束符（EOF），因此客户端使用NULL字节作为消息终止符。  
旧版本的客户端发送循环使用write_eof（）。

```python
    # This could be writer.writelines() except that
    # would make it harder to show each part of the message
    # being sent.
    for msg in messages:
        writer.write(msg)
        log.debug('sending {!r}'.format(msg))
    if writer.can_write_eof():
        writer.write_eof()
    await writer.drain()
```
新版本发送一个零字节（b'\x00'）。

```python
    # This could be writer.writelines() except that
    # would make it harder to show each part of the message
    # being sent.
    for msg in messages:
        writer.write(msg)
        log.debug('sending {!r}'.format(msg))
    # SSL does not support EOF, so send a null byte to indicate
    # the end of the message.
    writer.write(b'\x00')
    await writer.drain()
```
服务器中的echo（）协程必须查找NULL字节并在收到时关闭客户端连接。

```python
async def echo(reader, writer):
    address = writer.get_extra_info('peername')
    log = logging.getLogger('echo_{}_{}'.format(*address))
    log.debug('connection accepted')
    while True:
        data = await reader.read(128)
        terminate = data.endswith(b'\x00')
        data = data.rstrip(b'\x00')
        if data:
            log.debug('received {!r}'.format(data))
            writer.write(data)
            await writer.drain()
            log.debug('sent {!r}'.format(data))
        if not data or terminate:
            log.debug('message terminated, closing connection')
            writer.close()
            return
```
在一个窗口中运行服务器，在另一个窗口中运行客户端，生成此输出。

```
$ python3 asyncio_echo_server_ssl.py
asyncio: Using selector: KqueueSelector
main: starting up on localhost port 10000
echo_::1_53957: connection accepted
echo_::1_53957: received b'This is the message. '
echo_::1_53957: sent b'This is the message. '
echo_::1_53957: received b'It will be sent in parts.'
echo_::1_53957: sent b'It will be sent in parts.'
echo_::1_53957: message terminated, closing connection

$ python3 asyncio_echo_client_ssl.py
asyncio: Using selector: KqueueSelector
echo_client: connecting to localhost port 10000
echo_client: sending b'This is the message. '
echo_client: sending b'It will be sent '
echo_client: sending b'in parts.'
echo_client: waiting for response
echo_client: received b'This is the message. '
echo_client: received b'It will be sent in parts.'
echo_client: closing
main: closing event loop
```

### Interacting with Domain Name Services

应用程序使用网络与服务器进行通信，以进行域名服务（DNS）操作，例如在主机名和IP地址之间进行转换。asyncio在事件循环上有方便的方法来处理后台的那些操作，以避免在查询期间阻塞。

#### Address Lookup by Name

使用协程`getaddrinfo()`将主机名和端口号转换为IP或IPv6地址。
与socket模块中的函数版本一样，返回值是元组列表，该列表包含五条信息。

1. 地址族
2. 地址类型
3. 协议
4. 服务器的规范名称
5. 一个套接字地址元组，适用于在最初指定的端口上打开与服务器的连接

可以通过协议过滤查询，如本示例中所示，仅返回TCP响应。

```python
#asyncio_getaddrinfo.py
import asyncio
import logging
import socket
import sys

TARGETS = [
    ('pymotw.com', 'https'),
    ('doughellmann.com', 'https'),
    ('python.org', 'https'),
]

async def main(loop, targets):
    for target in targets:
        info = await loop.getaddrinfo(
            *target,
            proto=socket.IPPROTO_TCP,
        )

        for host in info:
            print('{:20}: {}'.format(target[0], host[4][0]))

event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main(event_loop, TARGETS))
finally:
    event_loop.close()
```
示例程序将主机名和协议名称转换为IP地址和端口号。

```
$ python3 asyncio_getaddrinfo.py

pymotw.com          : 66.33.211.242
doughellmann.com    : 66.33.211.240
python.org          : 23.253.135.79
python.org          : 2001:4802:7901::e60a:1375:0:6
```

#### Name Lookup by Address

协程`getnameinfo（）`以相反的方向工作，在可能的情况下将IP地址转换为主机名，将端口号转换为协议名称。

```python
#asyncio_getnameinfo.py
import asyncio
import logging
import socket
import sys

TARGETS = [
    ('66.33.211.242', 443),
    ('104.130.43.121', 443),
]

async def main(loop, targets):
    for target in targets:
        info = await loop.getnameinfo(target)
        print('{:15}: {} {}'.format(target[0], *info))

event_loop = asyncio.get_event_loop()
try:
    event_loop.run_until_complete(main(event_loop, TARGETS))
finally:
    event_loop.close()
```
此示例显示pymotw.com的IP地址指向DreamHost上的服务器，DreamHost是运行该站点的托管公司。检查的第二个IP地址是python.org，它不会解析回主机名。

```
$ python3 asyncio_getnameinfo.py

66.33.211.242  : apache2-echo.catalina.dreamhost.com https
104.130.43.121 : 104.130.43.121 https
```

### Working with Subprocesses

经常需要与其他程序和进程一起工作，以利用现有代码而无需重写代码或访问Python中不可用的库或功能。与网络I/O一样，asyncio包含两个用于启动另一个程序然后与之交互的抽象。

#### Using the Protocol Abstraction with Subprocesses

此示例使用协程启动进程以运行Unix命令df以查找本地磁盘上的可用空间。它使用subprocess_exec（）来启动进程并将其绑定到一个协议类，该协议类知道如何读取df命令输出并解析它。根据子进程的I/O事件自动调用协议类的方法。
由于stdin和stderr参数都设置为None，因此这些通信通道未连接到新进程。

```python
#asyncio_subprocess_protocol.py
import asyncio
import functools

async def run_df(loop):
    print('in run_df')

    cmd_done = asyncio.Future(loop=loop)
    factory = functools.partial(DFProtocol, cmd_done)
    proc = loop.subprocess_exec(
        factory,
        'df', '-hl',
        stdin=None,
        stderr=None,
    )
    try:
        print('launching process')
        transport, protocol = await proc
        print('waiting for process to complete')
        await cmd_done
    finally:
        transport.close()

    return cmd_done.result()
```
DFProtocol类派生自SubprocessProtocol，它定义了一个类的API用于通过管道与另一个进程通信。
done参数期望是一个Future，调用者用它监视进程的结束。

```python
class DFProtocol(asyncio.SubprocessProtocol):

    FD_NAMES = ['stdin', 'stdout', 'stderr']

    def __init__(self, done_future):
        self.done = done_future
        self.buffer = bytearray()
        super().__init__()
```
与套接字通信一样，在设置新进程的输入通道时会调用connection_made（）。transport参数是BaseSubprocessTransport的子类的实例。如果进程配置为接收输入，那么它可以读取进程输出的数据并写入数据到进程的输入流。

```python
    def connection_made(self, transport):
        print('process started {}'.format(transport.get_pid()))
        self.transport = transport
```
当进程生成输出时，将调用`pipe_data_received（）`，参数是发出数据的文件描述符以及从管道读取的实际数据。 协议类将进程的标准输出通道的输出保存在缓冲区中以供稍后处理。

```python
    def pipe_data_received(self, fd, data):
        print('read {} bytes from {}'.format(len(data), self.FD_NAMES[fd]))
        if fd == 1:
            self.buffer.extend(data)
```
当进程终止时，将调用`process_exited（）`。通过调用`get_returncode（）`，可以从传输对象获得进程的退出码。 在这种情况下，如果没有报告错误，则在通过该Future实例返回之前解码和解析可用输出。如果有错误，则假定结果为空。 设置future的result会告诉`run_df（）`进程已退出，所以它会清理然后返回结果。

```python
    def process_exited(self):
        print('process exited')
        return_code = self.transport.get_returncode()
        print('return code {}'.format(return_code))
        if not return_code:
            cmd_output = bytes(self.buffer).decode()
            results = self._parse_results(cmd_output)
        else:
            results = []
        self.done.set_result((return_code, results))
```
命令输出被解析为一系列字典，字典为每个输出行映射标题名称到他们的值，并返回结果列表。

```python
    def _parse_results(self, output):
        print('parsing results')
        # Output has one row of headers, all single words.  The
        # remaining rows are one per filesystem, with columns
        # matching the headers (assuming that none of the
        # mount points have whitespace in the names).
        if not output:
            return []
        lines = output.splitlines()
        headers = lines[0].split()
        devices = lines[1:]
        results = [ dict(zip(headers, line.split())) for line in devices ]
        return results
```
`run_df（）`协程使用`run_until_complete（）`运行，然后检查结果并打印每个设备上的可用空间。

```python
event_loop = asyncio.get_event_loop()
try:
    return_code, results = event_loop.run_until_complete(run_df(event_loop))
finally:
    event_loop.close()

if return_code:
    print('error exit {}'.format(return_code))
else:
    print('\nFree space:')
    for r in results:
        print('{Mounted:25}: {Avail}'.format(**r))
```
下面的输出显示了所执行步骤的顺序，以及运行它的系统上三个驱动器上的可用空间。

```
$ python3 asyncio_subprocess_protocol.py

in run_df
launching process
process started 49675
waiting for process to complete
read 332 bytes from stdout
process exited
return code 0
parsing results

Free space:
/                        : 233Gi
/Volumes/hubertinternal  : 157Gi
/Volumes/hubert-tm       : 2.3Ti
```

#### Calling Subprocesses with Coroutines and Streams

要使用协程直接运行进程，而不是通过Protocol子类访问它，请调用`create_subprocess_exec（）`并指定stdout，stderr和stdin中的哪一个连接到管道。协程产生子进程的结果是一个Process实例，可用于操作子进程或与之通信。

```python
#asyncio_subprocess_coroutine.py
import asyncio
import asyncio.subprocess

async def run_df():
    print('in run_df')

    buffer = bytearray()

    create = asyncio.create_subprocess_exec(
        'df', '-hl',
        stdout=asyncio.subprocess.PIPE,
    )
    print('launching process')
    proc = await create
    print('process started {}'.format(proc.pid))
```
在此示例中，df不需要除命令行参数之外的任何输入，因此下一步是读取所有输出。使用协议，无法控制一次读取多少数据。此示例使用readline（），但它也可以直接调用read（）来读取非面向行的数据。与协议示例一样，该命令的输出是缓冲的，因此可以在以后解析。

```python
    while True:
        line = await proc.stdout.readline()
        print('read {!r}'.format(line))
        if not line:
            print('no more output from command')
            break
        buffer.extend(line)
```
由于程序已完成，readline（）方法在没有更多输出时返回空字节字符串。为确保正确清理进程，下一步是等待进程完全退出。

```python
    print('waiting for process to complete')
    await proc.wait()
```
此时，可以检查退出状态以确定是解析输出还是处理错误，因为它没有产生输出。解析逻辑与前一个示例中的相同，但是处于一个独立函数（此处未显示），因为没有协议类可以将其隐藏。解析数据后，结果和退出码将返回给调用者。

```python
    return_code = proc.returncode
    print('return code {}'.format(return_code))
    if not return_code:
        cmd_output = bytes(buffer).decode()
        results = _parse_results(cmd_output)
    else:
        results = []

    return (return_code, results)
```
主程序看起来类似于基于协议的示例，因为实现的更改在`run_df（）`中是隔离的。

```python
event_loop = asyncio.get_event_loop()
try:
    return_code, results = event_loop.run_until_complete(run_df())
finally:
    event_loop.close()

if return_code:
    print('error exit {}'.format(return_code))
else:
    print('\nFree space:')
    for r in results:
        print('{Mounted:25}: {Avail}'.format(**r))
```
由于df的输出可以一次读取一行，因此可以回显显示程序的进度。 否则，输出看起来与前面的示例类似。

```
$ python3 asyncio_subprocess_coroutine.py

in run_df
launching process
process started 49678
read b'Filesystem     Size   Used  Avail Capacity   iused
ifree %iused  Mounted on\n'
read b'/dev/disk2s2  446Gi  213Gi  233Gi    48%  55955082
61015132   48%   /\n'
read b'/dev/disk1    465Gi  307Gi  157Gi    67%  80514922
41281172   66%   /Volumes/hubertinternal\n'
read b'/dev/disk3s2  3.6Ti  1.4Ti  2.3Ti    38% 181837749
306480579   37%   /Volumes/hubert-tm\n'
read b''
no more output from command
waiting for process to complete
return code 0
parsing results

Free space:
/                        : 233Gi
/Volumes/hubertinternal  : 157Gi
/Volumes/hubert-tm       : 2.3Ti
```

#### Sending Data to a Subprocess

前两个示例仅使用单个通信信道来从第二个进程读取数据。通常需要将数据发送到命令进行处理。此示例定义了一个协程，用于执行Unix命令tr以转换其输入流中的字符。在这个例子中，tr用于将小写字母转换为大写字母。  
to_upper（）协程将事件循环和输入字符串作为参数。它产生了第二个进程运行`tr [:lower:] [:upper:]`。

```python
#asyncio_subprocess_coroutine_write.py
import asyncio
import asyncio.subprocess

async def to_upper(input):
    print('in to_upper')

    create = asyncio.create_subprocess_exec(
        'tr', '[:lower:]', '[:upper:]',
        stdout=asyncio.subprocess.PIPE,
        stdin=asyncio.subprocess.PIPE,
    )
    print('launching process')
    proc = await create
    print('pid {}'.format(proc.pid))
```
接下来to_upper（）使用Process的communicate（）方法异步地将输入字符串发送到命令并读取所有结果输出。与这个方法的subprocess.Popen版本一样，communicate（）返回完整的输出字节字符串。如果一个命令可能产生的数据超出了可以很好地适应内存的数据，那么输入不能一次性产生，或者必须逐步处理输出，可以直接使用Process的stdin，stdout和stderr句柄而不是调用communicate（）。

```python
    print('communicating with process')
    stdout, stderr = await proc.communicate(input.encode())
```
I/O完成后，等待进程完全退出，确保正确清理。

```python
    print('waiting for process to complete')
    await proc.wait()
```
然后可以检查返回码，并解码输出字节串，以准备协程的返回值。

```python
    return_code = proc.returncode
    print('return code {}'.format(return_code))
    if not return_code:
        results = bytes(stdout).decode()
    else:
        results = ''

    return (return_code, results)
```
程序的主要部分建立一个要转换的消息字符串，然后设置事件循环以运行to_upper（）并打印结果。

```python
MESSAGE = """
This message will be converted
to all caps.
"""

event_loop = asyncio.get_event_loop()
try:
    return_code, results = event_loop.run_until_complete(to_upper(MESSAGE))
finally:
    event_loop.close()

if return_code:
    print('error exit {}'.format(return_code))
else:
    print('Original: {!r}'.format(MESSAGE))
    print('Changed : {!r}'.format(results))
```
输出显示操作序列，然后显示简单文本消息是如何转换的。

```
$ python3 asyncio_subprocess_coroutine_write.py

in to_upper
launching process
pid 49684
communicating with process
waiting for process to complete
return code 0
Original: '\nThis message will be converted\nto all caps.\n'
Changed : '\nTHIS MESSAGE WILL BE CONVERTED\nTO ALL CAPS.\n'
```

### Receiving Unix Signals

Unix系统事件通知通常会中断应用程序，触发其处理程序。与asyncio一起使用时，信号处理程序回调与事件循环管理的其他协程和回调交错。结果是更少的函数被中断，以及需要提供安全防护来清理不完整的操作。  
信号处理程序必须是常规的callables，而不是coroutines。

```python
#asyncio_signal.py
import asyncio
import functools
import os
import signal

def signal_handler(name):
    print('signal_handler({!r})'.format(name))
```
使用`add_signal_handler（）`注册信号处理程序。第一个参数是信号，第二个参数是回调。 回调没有参数传递，因此如果需要参数，可以使用functools.partial（）包装函数。

```python
event_loop = asyncio.get_event_loop()

event_loop.add_signal_handler(
    signal.SIGHUP,
    functools.partial(signal_handler, name='SIGHUP'),
)
event_loop.add_signal_handler(
    signal.SIGUSR1,
    functools.partial(signal_handler, name='SIGUSR1'),
)
event_loop.add_signal_handler(
    signal.SIGINT,
    functools.partial(signal_handler, name='SIGINT'),
)
```
这个示例程序使用协程通过os.kill（）向自己发送信号。在发送每个信号之后，协程放弃控制权以允许处理程序运行。 在正常的应用程序中，应用程序代码返回控制权到事件循环的位置会更多，并且不需要像这样人为的让出控制权。

```python
async def send_signals():
    pid = os.getpid()
    print('starting send_signals for {}'.format(pid))

    for name in ['SIGHUP', 'SIGHUP', 'SIGUSR1', 'SIGINT']:
        print('sending {}'.format(name))
        os.kill(pid, getattr(signal, name))
        # Yield control to allow the signal handler to run,
        # since the signal does not interrupt the program
        # flow otherwise.
        print('yielding control')
        await asyncio.sleep(0.01)
    return
```
主程序运行send_signals（）直到它发送了所有信号。

```python
try:
    event_loop.run_until_complete(send_signals())
finally:
    event_loop.close()
```
输出显示send_signals（）在发送信号让出控制权后处理程序如何被调用。

```
$ python3 asyncio_signal.py

starting send_signals for 21772
sending SIGHUP
yielding control
signal_handler('SIGHUP')
sending SIGHUP
yielding control
signal_handler('SIGHUP')
sending SIGUSR1
yielding control
signal_handler('SIGUSR1')
sending SIGINT
yielding control
signal_handler('SIGINT')
```

### Combining Coroutines with Threads and Processes

很多现有的库都无法原生的与asyncio一起使用。它们可能会阻塞或依赖于模块无法提供的并发功能。通过使用concurrent.futures中的执行器在单独的线程或单独的进程中运行代码，仍然可以在基于asyncio的应用程序中使用这些库。

#### Threads

事件循环的`run_in_executor（）`方法接受一个执行器实例，一个要调用的常规callable，以及要传递给callable的任何参数。它返回一个Future，可用于等待函数完成其工作并返回一些东西。如果没有传入执行器，则创建ThreadPoolExecutor。此示例显式创建一个执行器，以限制它具有的可用工作线程数。

ThreadPoolExecutor启动其工作线程，然后在线程中调用每个提供的函数一次。此示例显示如何组合`run_in_executor（）`和`wait（）`以当阻塞函数运行在独立的线程中时，让协程把控制权返回给事件循环，然后在这些函数完成时唤醒协程。

```python
#asyncio_executor_thread.py
import asyncio
import concurrent.futures
import logging
import sys
import time

def blocks(n):
    log = logging.getLogger('blocks({})'.format(n))
    log.info('running')
    time.sleep(0.1)
    log.info('done')
    return n ** 2

async def run_blocking_tasks(executor):
    log = logging.getLogger('run_blocking_tasks')
    log.info('starting')

    log.info('creating executor tasks')
    loop = asyncio.get_event_loop()
    blocking_tasks = [
        loop.run_in_executor(executor, blocks, i)
        for i in range(6)
    ]
    log.info('waiting for executor tasks')
    completed, pending = await asyncio.wait(blocking_tasks)
    results = [t.result() for t in completed]
    log.info('results: {!r}'.format(results))

    log.info('exiting')

if __name__ == '__main__':
    # Configure logging to show the name of the thread
    # where the log message originates.
    logging.basicConfig(
        level=logging.INFO,
        format='%(threadName)10s %(name)18s: %(message)s',
        stream=sys.stderr,
    )

    # Create a limited thread pool.
    executor = concurrent.futures.ThreadPoolExecutor(
        max_workers=3,
    )

    event_loop = asyncio.get_event_loop()
    try:
        event_loop.run_until_complete(run_blocking_tasks(executor))
    finally:
        event_loop.close()
```
`asyncio_executor_thread.py`使用logging来方便地指示生成每条日志消息的线程和函数。因为在对blocks（）的每次调用中使用单独的记录器，所以输出清楚地显示了被重用的相同线程使用不同的参数调用该函数的多个副本。

```
$ python3 asyncio_executor_thread.py

MainThread run_blocking_tasks: starting
MainThread run_blocking_tasks: creating executor tasks
ThreadPoolExecutor-0_0          blocks(0): running
ThreadPoolExecutor-0_1          blocks(1): running
ThreadPoolExecutor-0_2          blocks(2): running
MainThread run_blocking_tasks: waiting for executor tasks
ThreadPoolExecutor-0_0          blocks(0): done
ThreadPoolExecutor-0_1          blocks(1): done
ThreadPoolExecutor-0_2          blocks(2): done
ThreadPoolExecutor-0_0          blocks(3): running
ThreadPoolExecutor-0_1          blocks(4): running
ThreadPoolExecutor-0_2          blocks(5): running
ThreadPoolExecutor-0_0          blocks(3): done
ThreadPoolExecutor-0_2          blocks(5): done
ThreadPoolExecutor-0_1          blocks(4): done
MainThread run_blocking_tasks: results: [0, 9, 16, 25, 1, 4]
MainThread run_blocking_tasks: exiting
```

#### Processes

ProcessPoolExecutor的工作方式大致相同，创建一组工作进程而不是线程。使用单独的进程需要更多的系统资源，但对于计算密集型操作，在每个CPU核心上运行单独的任务是有意义的。

```python
#asyncio_executor_process.py
# changes from asyncio_executor_thread.py

if __name__ == '__main__':
    # Configure logging to show the id of the process
    # where the log message originates.
    logging.basicConfig(
        level=logging.INFO,
        format='PID %(process)5s %(name)18s: %(message)s',
        stream=sys.stderr,
    )

    # Create a limited process pool.
    executor = concurrent.futures.ProcessPoolExecutor(
        max_workers=3,
    )

    event_loop = asyncio.get_event_loop()
    try:
        event_loop.run_until_complete(
            run_blocking_tasks(executor)
        )
    finally:
        event_loop.close()
```
从线程转移到进程所需的唯一更改是创建不同类型的执行器。此示例还更改日志记录格式字符串以包括进程ID而不是线程名称，以证明任务实际上在单独的进程中运行。

```
$ python3 asyncio_executor_process.py

PID 40498 run_blocking_tasks: starting
PID 40498 run_blocking_tasks: creating executor tasks
PID 40498 run_blocking_tasks: waiting for executor tasks
PID 40499          blocks(0): running
PID 40500          blocks(1): running
PID 40501          blocks(2): running
PID 40499          blocks(0): done
PID 40500          blocks(1): done
PID 40501          blocks(2): done
PID 40500          blocks(3): running
PID 40499          blocks(4): running
PID 40501          blocks(5): running
PID 40499          blocks(4): done
PID 40500          blocks(3): done
PID 40501          blocks(5): done
PID 40498 run_blocking_tasks: results: [1, 4, 9, 0, 16, 25]
PID 40498 run_blocking_tasks: exiting
```

### Debugging with asyncio

asyncio内置了几个有用的调试功能。  
首先，事件循环使用logging在运行时发出状态消息。如果在应用程序中启用了日志记录，则其中一些可用。可以通过告诉循环发出更多调试消息来开启其他的。调用set_debug（）传递一个布尔值，指示是否应启用调试。  
因为基于asyncio的应用程序对贪婪的协程释放控制权失败非常敏感，所以支持检测事件循环中内置的慢（slow）回调。通过启用调试打开它，并通过将循环的`slow_callback_duration`属性设置为在“多少秒”后发出警告的秒数来控制“慢”的定义。  
最后，如果使用asyncio的应用程序在不清除某些协程或其他资源的情况下退出，则可能意味着存在逻辑错误阻止某些应用程序代码的运行。启用ResourceWarning警告会导致在程序退出时报告这些情况。

```python
#asyncio_debug.py
import argparse
import asyncio
import logging
import sys
import time
import warnings

parser = argparse.ArgumentParser('debugging asyncio')
parser.add_argument(
    '-v',
    dest='verbose',
    default=False,
    action='store_true',
)
args = parser.parse_args()

logging.basicConfig(
    level=logging.DEBUG,
    format='%(levelname)7s: %(message)s',
    stream=sys.stderr,
)
LOG = logging.getLogger('')

async def inner():
    LOG.info('inner starting')
    # Use a blocking sleep to simulate
    # doing work inside the function.
    time.sleep(0.1)
    LOG.info('inner completed')

async def outer(loop):
    LOG.info('outer starting')
    await asyncio.ensure_future(loop.create_task(inner()))
    LOG.info('outer completed')

event_loop = asyncio.get_event_loop()
if args.verbose:
    LOG.info('enabling debugging')

    # Enable debugging
    event_loop.set_debug(True)

    # Make the threshold for "slow" tasks very very small for
    # illustration. The default is 0.1, or 100 milliseconds.
    event_loop.slow_callback_duration = 0.001

    # Report all mistakes managing asynchronous resources.
    warnings.simplefilter('always', ResourceWarning)

LOG.info('entering event loop')
event_loop.run_until_complete(outer(event_loop))
```
在未启用调试的情况下运行时，此应用程序的一切看起来都正常

```
$ python3 asyncio_debug.py

  DEBUG: Using selector: KqueueSelector
   INFO: entering event loop
   INFO: outer starting
   INFO: inner starting
   INFO: inner completed
   INFO: outer completed
```
打开调试会暴露它所遇到的一些问题，包括尽管inner（）完成，但它比设置的`slow_callback_duration`需要更多的时间，并且当程序退出时事件循环没有正确关闭。

```
$ python3 asyncio_debug.py -v

  DEBUG: Using selector: KqueueSelector
   INFO: enabling debugging
   INFO: entering event loop
   INFO: outer starting
   INFO: inner starting
   INFO: inner completed
WARNING: Executing <Task finished coro=<inner() done, defined at
asyncio_debug.py:33> result=None created at asyncio_debug.py:43>
took 0.103 seconds
   INFO: outer completed
```

## concurrent.futures — Manage Pools of Concurrent Tasks

目的：轻松管理并发和并行运行的任务。

concurrent.futures模块提供了使用线程或进程 worker池运行任务的接口。API是相同的，因此应用程序可以在最小的更改之间切换线程和进程。该模块提供了两种类型的类，用于与池进行交互。Executors（执行器）用于管理worker池，而futures用于管理workers计算的结果。要使用worker池，应用程序将创建适当的执行器类的实例，然后提交任务给它以使其运行。每个任务启动时，都会返回Future实例。当需要任务的结果时，应用程序可以使用Future来阻塞，直到结果可用。提供了各种API以便于等待任务完成，因此不需要直接管理Future对象。

### Using map() with a Basic Thread Pool

ThreadPoolExecutor管理一组工作线程，在它们可用于更多工作时将任务传递给它们。此示例使用map（）从输入iterable同时生成一组结果。该任务使用time.sleep（）暂停不同的时间来说明：无论并发任务的执行顺序如何，map（）始终根据输入按顺序返回值。

```python
#futures_thread_pool_map.py
from concurrent import futures
import threading
import time

def task(n):
    print('{}: sleeping {}'.format(threading.current_thread().name, n) )
    time.sleep(n / 10)
    print('{}: done with {}'.format(threading.current_thread().name, n) )
    return n / 10

ex = futures.ThreadPoolExecutor(max_workers=2)
print('main: starting')
results = ex.map(task, range(5, 0, -1))
print('main: unprocessed results {}'.format(results))
print('main: waiting for real results')
real_results = list(results)
print('main: results: {}'.format(real_results))
```
map（）的返回值实际上是一种特殊类型的迭代器，它会在主程序迭代它时等待每个响应。

```
$ python3 futures_thread_pool_map.py

main: starting
ThreadPoolExecutor-0_0: sleeping 5
ThreadPoolExecutor-0_1: sleeping 4
main: unprocessed results <generator object Executor.map.<locals>.result_iterator at 0x103e12780>
main: waiting for real results
ThreadPoolExecutor-0_1: done with 4
ThreadPoolExecutor-0_1: sleeping 3
ThreadPoolExecutor-0_0: done with 5
ThreadPoolExecutor-0_0: sleeping 2
ThreadPoolExecutor-0_0: done with 2
ThreadPoolExecutor-0_0: sleeping 1
ThreadPoolExecutor-0_1: done with 3
ThreadPoolExecutor-0_0: done with 1
main: results: [0.5, 0.4, 0.3, 0.2, 0.1]
```

### Scheduling Individual Tasks

除了使用map（）之外，执行器还可以使用submit（）调度单个任务，并使用返回的Future实例等待该任务的结果。

```python
#futures_thread_pool_submit.py
from concurrent import futures
import threading
import time

def task(n):
    print('{}: sleeping {}'.format(threading.current_thread().name, n) )
    time.sleep(n / 10)
    print('{}: done with {}'.format(threading.current_thread().name, n) )
    return n / 10

ex = futures.ThreadPoolExecutor(max_workers=2)
print('main: starting')
f = ex.submit(task, 5)
print('main: future: {}'.format(f))
print('main: waiting for results')
result = f.result()
print('main: result: {}'.format(result))
print('main: future after result: {}'.format(f))
```
任务完成后future的状态会发生变化并使result可用。

```
$ python3 futures_thread_pool_submit.py

main: starting
ThreadPoolExecutor-0_0: sleeping 5
main: future: <Future at 0x1034e1ef0 state=running>
main: waiting for results
ThreadPoolExecutor-0_0: done with 5
main: result: 0.5
main: future after result: <Future at 0x1034e1ef0 state=finished returned float>
```

### Waiting for Tasks in Any Order

调用Future的result（）方法阻塞，直到任务完成（通过返回值或引发异常），或者被取消。可以使用map（）按任务被调度的顺序访问多个任务的结果。如果处理result的顺序无关紧要，请使用as_completed（）在每个任务完成时处理它们。

```python
#futures_as_completed.py
from concurrent import futures
import random
import time

def task(n):
    time.sleep(random.random())
    return (n, n / 10)

ex = futures.ThreadPoolExecutor(max_workers=5)
print('main: starting')

wait_for = [ ex.submit(task, i) for i in range(5, 0, -1) ]

for f in futures.as_completed(wait_for):
    print('main: result: {}'.format(f.result()))
```
由于池具有与任务一样多的workers，因此可以启动所有任务。它们以随机顺序完成，因此每次运行示例时，as_completed（）生成的值都不同。

```
$ python3 futures_as_completed.py

main: starting
main: result: (1, 0.1)
main: result: (5, 0.5)
main: result: (3, 0.3)
main: result: (2, 0.2)
main: result: (4, 0.4)
```

### Future Callbacks

要在任务完成时执行某些操作，而不显式等待结果，请使用`add_done_callback（）`指定在Future完成时调用的新函数。 
回调应该是一个可调用对象，带有单个参数即Future实例。

```python
#futures_future_callback.py
from concurrent import futures
import time

def task(n):
    print('{}: sleeping'.format(n))
    time.sleep(0.5)
    print('{}: done'.format(n))
    return n / 10

def done(fn):
    if fn.cancelled():
        print('{}: canceled'.format(fn.arg))
    elif fn.done():
        error = fn.exception()
        if error:
            print('{}: error returned: {}'.format(fn.arg, error))
        else:
            result = fn.result()
            print('{}: value returned: {}'.format(fn.arg, result))

if __name__ == '__main__':
    ex = futures.ThreadPoolExecutor(max_workers=2)
    print('main: starting')
    f = ex.submit(task, 5)
    f.arg = 5
    f.add_done_callback(done)
    result = f.result()
```
无论Future被认为“完成”的原因是什么，都会调用回调，因此有必要检查传入回调的对象（future）的状态在以任何方式使用它之前。

```
$ python3 futures_future_callback.py

main: starting
5: sleeping
5: done
5: value returned: 0.5
```

### Canceling Tasks

如果已经提交但未启动，则可以通过调用其cancel（）方法取消Future。

```python
#futures_future_callback_cancel.py
from concurrent import futures
import time

def task(n):
    print('{}: sleeping'.format(n))
    time.sleep(0.5)
    print('{}: done'.format(n))
    return n / 10

def done(fn):
    if fn.cancelled():
        print('{}: canceled'.format(fn.arg))
    elif fn.done():
        print('{}: not canceled'.format(fn.arg))

if __name__ == '__main__':
    ex = futures.ThreadPoolExecutor(max_workers=2)
    print('main: starting')
    tasks = []

    for i in range(10, 0, -1):
        print('main: submitting {}'.format(i))
        f = ex.submit(task, i)
        f.arg = i
        f.add_done_callback(done)
        tasks.append((i, f))

    for i, t in reversed(tasks):
        if not t.cancel():
            print('main: did not cancel {}'.format(i))

    ex.shutdown()
```
cancel（）返回一个布尔值，指示该任务是否能够被取消。

```
$ python3 futures_future_callback_cancel.py

main: starting
main: submitting 10
10: sleeping
main: submitting 9
9: sleeping
main: submitting 8
main: submitting 7
main: submitting 6
main: submitting 5
main: submitting 4
main: submitting 3
main: submitting 2
main: submitting 1
1: canceled
2: canceled
3: canceled
4: canceled
5: canceled
6: canceled
7: canceled
8: canceled
main: did not cancel 9
main: did not cancel 10
10: done
10: not canceled
9: done
9: not canceled
```

### Exceptions in Tasks

如果任务引发未处理的异常，则会将其保存到任务的Future中，并通过result（）或exception（）方法访问。

```python
#futures_future_exception.py
from concurrent import futures

def task(n):
    print('{}: starting'.format(n))
    raise ValueError('the value {} is no good'.format(n))

ex = futures.ThreadPoolExecutor(max_workers=2)
print('main: starting')
f = ex.submit(task, 5)

error = f.exception()
print('main: error: {}'.format(error))

try:
    result = f.result()
except ValueError as e:
    print('main: saw error "{}" when accessing result'.format(e))
```
如果在任务函数中引发未处理的异常后调用result（），则会在当前上下文中重新引发相同的异常。

```
$ python3 futures_future_exception.py

main: starting
5: starting
main: error: the value 5 is no good
main: saw error "the value 5 is no good" when accessing result
```

### Context Manager

执行器作为上下文管理器，同时运行任务并等待它们全部完成。当上下文管理器退出时，执行器的shutdown（）方法被调用。

```python
#futures_context_manager.py
from concurrent import futures

def task(n):
    print(n)

with futures.ThreadPoolExecutor(max_workers=2) as ex:
    print('main: starting')
    ex.submit(task, 1)
    ex.submit(task, 2)
    ex.submit(task, 3)
    ex.submit(task, 4)

print('main: done')
```
当执行离开当前作用域要清除线程或进程资源时，使用执行器的这种模式很有用。

```
$ python3 futures_context_manager.py

main: starting
1
2
3
4
main: done
```

### Process Pools

ProcessPoolExecutor的工作方式与ThreadPoolExecutor相同，但使用进程而不是线程。 这允许CPU密集型操作使用单独的CPU，而不会被CPython解释器的全局解释器锁阻塞。

```python
#futures_process_pool_map.py
from concurrent import futures
import os

def task(n):
    return (n, os.getpid())

ex = futures.ProcessPoolExecutor(max_workers=2)
results = ex.map(task, range(5, 0, -1))
for n, pid in results:
    print('ran task {} in process {}'.format(n, pid))
```
与线程池一样，单个工作进程可以重用于多个任务。

```
$ python3 futures_process_pool_map.py

ran task 5 in process 40854
ran task 4 in process 40854
ran task 3 in process 40854
ran task 2 in process 40854
ran task 1 in process 40854
```

如果其中一个工作进程发生某些事情导致其意外退出，则ProcessPoolExecutor被视为“已损坏（broken）”，并且将不再调度任务。

```python
#futures_process_pool_broken.py
from concurrent import futures
import os
import signal

with futures.ProcessPoolExecutor(max_workers=2) as ex:
    print('getting the pid for one worker')
    f1 = ex.submit(os.getpid)
    pid1 = f1.result()

    print('killing process {}'.format(pid1))
    os.kill(pid1, signal.SIGHUP)

    print('submitting another task')
    f2 = ex.submit(os.getpid)
    try:
        pid2 = f2.result()
    except futures.process.BrokenProcessPool as e:
        print('could not start new tasks: {}'.format(e))
```
处理result时实际抛出BrokenProcessPool异常，而不是在提交新任务时抛出异常。

```
$ python3 futures_process_pool_broken.py

getting the pid for one worker
killing process 40858
submitting another task
could not start new tasks: A process in the process pool was
terminated abruptly while the future was running or pending.
```
