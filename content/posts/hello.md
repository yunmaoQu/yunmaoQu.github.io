---
title: "Hello"
date: 2024-09-17T19:03:15+08:00
draft: false
---





# hello world!!!


#!/usr/bin/env python3
# @Function
# Find out the highest cpu consumed threads of java processes, and print the stack of these threads.
#
# @Usage
#   $ show-busy-java-threads
# 从所有运行的Java进程中找出最消耗CPU的线程（缺省5个），打印出其线程栈

# 缺省会自动从所有的Java进程中找出最消耗CPU的线程，这样用更方便
# 当然你可以通过 -p 选项 手动指定要分析的Java进程Id，以保证只会显示你关心的那个Java进程的信息
#   $ show-busy-java-threads -p <指定的Java进程Id>
#   $ show-busy-java-threads -p 42
#   $ show-busy-java-threads -p 42,47

#   $ show-busy-java-threads -c <要展示示的线程栈个数>

#   $ show-busy-java-threads <重复执行的间隔秒数> [<重复执行的次数>]
#   多次执行；这2个参数的使用方式类似vmstat命令

#   $ show-busy-java-threads -a <运行输出的记录到的文件>
# 记录到文件以方便回溯查看

#   $ show-busy-java-threads -S <存储jstack输出文件的目录>
# 指定jstack输出文件的存储目录，方便记录以后续分析
#
# @online-doc https://github.com/qingfeng503/useful-scripts/blob/dev-3.x/docs/java.md#-show-busy-java-threads# @Function
# Find out the highest cpu consumed threads of java processes, and print the stack of these threads.
#
__author__ = 'yunmaoQu'
import os
import subprocess
import time
import uuid
import signal
from datetime import datetime
import sys
import argparse
import shutil


# Global Variables
PROG = os.path.basename(__file__)
PROG_VERSION = '3.x-dev'
COMMAND_LINE = [sys.argv[0]] + sys.argv[1:]

# Get user and system information
WHOAMI = os.getenv('USER') or subprocess.getoutput('whoami')
UNAME = os.uname().sysname

NL = '\n'  # New line character

# Utility Functions
def color_output(color_code, message):
    """
    Print message with color if stdout is a terminal.
    """
    if sys.stdout.isatty():
        print(f"\033[1;{color_code}m{message}\033[0m")
    else:
        print(message)
def normal_output(message):
    """
    Print normal output and append to file if necessary.
    """
    print(message)
    append_to_file(message, append_file=append_file, store_dir=store_dir)

def red_output(message):
    color_output(31, message)

def green_output(message):
    color_output(32, message)

def yellow_output(message):
    color_output(33, message)

def blue_output(message):
    color_output(36, message)


def append_to_file(message, append_file=None, store_dir=None):
    """
    Append the given message to the specified file or to the store file.
    """
    if append_file and os.access(append_file, os.W_OK):
        with open(append_file, 'a') as f:
            f.write(message + '\n')
    if store_dir:
        store_file = os.path.join(store_dir, f"{PROG}_log")
        with open(store_file, 'a') as f:
            f.write(message + '\n')

def log_and_run(command):
    """
    Log the command and run it.
    """
    print(f"{command}")
    print()
    subprocess.run(command, shell=True)

def is_non_negative_float_number(value):
    """
    Check if the value is a non-negative float.
    """
    try:
        return float(value) >= 0
    except ValueError:
        return False

def is_natural_number(value):
    """
    Check if the value is a natural number (positive integer).
    """
    try:
        return int(value) >= 0
    except ValueError:
        return False


def is_natural_number_list(value_list):
    return all(is_natural_number(value) for value in value_list.split(','))

def print_calling_command_line(args):
    """
    Print the full calling command with the arguments.
    """
    return ' '.join(map(str, args))

def die(prompt_help=False, exit_status=2, *args):
    # Print the error message if any arguments are provided
    if args:
        colorPrint("1;31", f"{PROG}: {' '.join(args)}")
    
    # If prompt_help is True, print the help message line
    if prompt_help:
        print(f"Try '{PROG} --help' for more information.")
    
    # Exit with the specified exit status
    sys.exit(exit_status)
def usage():
    usage_text = f"""\
Usage: {PROG} [OPTION]... [delay [count]]
Find out the highest CPU consumed threads of Java processes,
and print the stack of these threads.

Example:
  {PROG}       # Show busy Java threads info
  {PROG} 1     # Update every 1 second, (stop by e.g., CTRL+C)
  {PROG} 3 10  # Update every 3 seconds, update 10 times

Output control:
  -p, --pid <java pid(s)>   Find out the highest CPU consumed threads from
                            the specified Java process.
                            Support pid list (e.g., 42,47).
                            Default from all Java process.
  -c, --count <num>         Set the thread count to show, default is 5.
                            Set count 0 to show all threads.
  -a, --append-file <file>  Specifies the file to append output as log.
  -S, --store-dir <dir>     Specifies the directory for storing
                            the intermediate files, and keep files.
                            Default store intermediate files at tmp dir,
                            and auto remove after run. Use this option to keep
                            files so as to review jstack/top/ps output later.
  delay                     The delay between updates in seconds.
  count                     The number of updates.
                            delay/count arguments imitate the style of
                            the vmstat command.

jstack control:
  -s, --jstack-path <path>  Specifies the path of jstack command.
  -F, --force               Set jstack to force a thread dump.
                            Use when jstack does not respond (process is hung).
  -m, --mix-native-frames   Set jstack to print both Java and
                            native frames (mixed mode).
  -l, --lock-info           Set jstack with long listing.
                            Prints additional information about locks.

CPU usage calculation control:
  -i, --cpu-sample-interval Specifies the delay between CPU samples to get
                            thread CPU usage percentage during this interval.
                            Default is 0.5 (second).
                            Set interval 0 to get the percentage of time spent
                            running during the *entire lifetime* of a process.

Miscellaneous:
  -h, --help                Display this help and exit.
  -V, --version             Display version information and exit.
"""
    print(usage_text)
    sys.exit()

def prog_version():
    """
    Print the program version.
    """
    print(f"{PROG} {PROG_VERSION}")
    sys.exit()


# Check if the operating system is Linux
import platform
if platform.system() != "Linux":
    die("only support Linux!")
# Default values
count = 5
cpu_sample_interval = 0.5
force = None
mix_native_frames = None
lock_info = None
pid_list = None
append_file = None
store_dir = None
# Argument parsing using argparse
parser = argparse.ArgumentParser(description="Script to demonstrate argument parsing and validation.",add_help=False)
parser.add_argument("-c", "--count", type=int, default=5, help="Set count value (default: 5)")
parser.add_argument("-p", "--pid", type=str, help="Set pid list")
parser.add_argument("-a", "--append-file", type=str, help="Set append file")
parser.add_argument("-s", "--jstack-path", type=str, help="Set jstack path")
parser.add_argument("-S", "--store-dir", type=str, help="Set store directory")
parser.add_argument("-i", "--cpu-sample-interval", type=float, default=0.5, help="Set CPU sample interval (default: 0.5)")
parser.add_argument("-P", "--use-ps", action="store_true", help="Use PS (sets CPU sample interval to 0)")
parser.add_argument("-d", "--top-delay", type=float, help="Set top delay")
parser.add_argument("-F", "--force", action="store_true", help="Use force")
parser.add_argument("-m", "--mix-native-frames", action="store_true", help="Use mix native frames")
parser.add_argument("-l", "--lock-info", action="store_true", help="Use lock info")
parser.add_argument("-h", "--help", action="store_true", help="Show help")
parser.add_argument("-V", "--version", action="store_true", help="Show version")

args = parser.parse_args()

# Set cpu_sample_interval to 0 if --use-ps is specified
if args.use_ps:
    args.cpu_sample_interval = 0

# Validate cpu_sample_interval
if not is_non_negative_float_number(args.cpu_sample_interval):
    die(f"CPU sample interval ({args.cpu_sample_interval}) is not a non-negative float number!")

# Validate update_delay and update_count
update_delay = float(sys.argv[1]) if len(sys.argv) > 1 else 0
if not is_non_negative_float_number(update_delay):
    die(f"Update delay ({update_delay}) is not a non-negative float number!")

update_count = int(sys.argv[2]) if len(sys.argv) > 2 else 0
if not is_natural_number(update_count):
    die(f"Update count ({update_count}) is not a natural number!")

# Validate pid_list
if args.pid:
    args.pid = args.pid.replace(" ", "")
    if not is_natural_number_list(args.pid):
        die(f"PIDs ({args.pid}) are illegal! Example: 42 or 42,99,67")

# Check the directory of append-file mode, create if not existed
if args.append_file:
    append_file_dir = os.path.dirname(args.append_file)
    if os.path.exists(args.append_file):
        if not os.path.isfile(args.append_file):
            die(f"{args.append_file} (specified by option -a, for storing run output files) exists but is not a file!")
        if not os.access(args.append_file, os.W_OK):
            die(f"File {args.append_file} (specified by option -a, for storing run output files) exists but is not writable!")
    elif not os.path.exists(append_file_dir):
        os.makedirs(append_file_dir, exist_ok=True)
    elif not os.path.isdir(append_file_dir):
        die(f"Directory {append_file_dir} (specified by option -a, for storing run output files) exists but is not a directory!")
    elif not os.access(append_file_dir, os.W_OK):
        die(f"Directory {append_file_dir} (specified by option -a, for storing run output files) exists but is not writable!")

# Check store directory mode, create directory if not existed
if args.store_dir:
    if os.path.exists(args.store_dir):
        if not os.path.isdir(args.store_dir):
            die(f"{args.store_dir} (specified by option -S, for storing output files) exists but is not a directory!")
        if not os.access(args.store_dir, os.W_OK):
            die(f"Directory {args.store_dir} (specified by option -S, for storing output files) exists but is not writable!")
    else:
        os.makedirs(args.store_dir, exist_ok=True)

def is_executable(file_path):
    return os.path.isfile(file_path) and os.access(file_path, os.X_OK)
jstack_path = args.jstack_path

# 1. Check if jstack_path is set by -s option
if jstack_path:  # Assuming jstack_path is already set from argparse or other logic
    if not is_executable(jstack_path):
        die(f"{jstack_path} (set by -s option) is NOT found or NOT executable!", hint=True)

# 2. Search for jstack under JAVA_HOME
elif 'JAVA_HOME' in os.environ:
    java_home = os.environ['JAVA_HOME']
    potential_jstack_paths = [
        os.path.join(java_home, 'bin', 'jstack'),
        os.path.join(java_home, '..', 'bin', 'jstack')
    ]

    for path in potential_jstack_paths:
        if os.path.isfile(path):
            if not is_executable(path):
                die(f"found {path} is NOT executable!{NL}Use -s option to set jstack path manually.", hint=True)
            jstack_path = path
            break
    else:
        die(f"jstack NOT found under $JAVA_HOME/bin or $JAVA_HOME/../bin!{NL}Use -s option to set jstack path manually.", hint=True)

# 3. Search for jstack in the PATH
else:
    try:
        jstack_path = subprocess.check_output(['which', 'jstack']).strip().decode('utf-8')
        if not is_executable(jstack_path):
            die(f"found {jstack_path} from PATH is NOT executable!{NL}Use -s option to set jstack path manually.", hint=True)
    except subprocess.CalledProcessError:
        die(f"jstack NOT found by JAVA_HOME ({os.environ.get('JAVA_HOME', 'not set')}) setting and PATH!{NL}Use -s option to set jstack path manually.", hint=True)
    except Exception:
        die(f"jstack NOT found in PATH!{NL}Use -s option to set jstack path manually.", hint=True)

# Mark jstack_path as readonly (conceptual, Python doesn't have true readonly)
print(f"Using jstack path: {jstack_path}")

# Generate a unique identifier for the session
run_timestamp = datetime.now().strftime("%Y-%m-%d_%H:%M:%S.%f")
uuid_str = f"{PROG}_{run_timestamp}_{os.getpid()}_{uuid.uuid4().hex}"

tmp_store_dir = f"/tmp/{uuid_str}"
store_file_prefix = f"{tmp_store_dir}/{run_timestamp}_"
if store_dir:
    store_file_prefix = f"{store_dir}/{run_timestamp}_"

os.makedirs(tmp_store_dir, exist_ok=True)

def cleanup_when_exit():
    """Cleanup temporary directories on exit."""
    if os.path.exists(tmp_store_dir):
        subprocess.call(['rm', '-rf', tmp_store_dir])

signal.signal(signal.SIGTERM, lambda signum, frame: cleanup_when_exit())
signal.signal(signal.SIGINT, lambda signum, frame: cleanup_when_exit())

def head_info(timestamp, update_round_num):
    """Print header information."""
    print("=" * 80)
    print(f"{timestamp} [{update_round_num + 1}/{update_count}]: {print_calling_command_line()}")
    print("=" * 80)
    print()

def print_calling_command_line():
    """Simulate a function to print the calling command line."""
    # In Python, sys.argv can be used to print the command line args
    import sys
    return ' '.join(sys.argv)

def die(message):
    """Exit the program with an error message."""
    print(f"Error: {message}")
    exit(1)

def find_busy_java_threads_by_ps():
    """Use `ps` to find busy Java threads (by CPU usage)."""
    ps_process_select_options = f"-p {pid_list}" if pid_list else "-C java -C jsvc"
    ps_cmd_line = f"ps {ps_process_select_options} -wwLo 'pid,lwp,pcpu,user' --no-headers"

    try:
        ps_out = subprocess.check_output(ps_cmd_line, shell=True).decode()
        sorted_ps_out = "\n".join(sorted(ps_out.splitlines(), key=lambda x: float(x.split()[2]), reverse=True))

        if store_dir:
            with open(f"{store_file_prefix}{update_round_num + 1}_ps", 'w') as f:
                f.write(ps_cmd_line + "\n" + sorted_ps_out)

        if count > 0:
            return "\n".join(sorted_ps_out.splitlines()[:count])
        else:
            return sorted_ps_out

    except subprocess.CalledProcessError:
        die("No Java process found!")

def find_busy_java_threads_by_top():
    """Use `top` to find busy Java threads (by CPU usage)."""
    ps_process_select_options = f"-p {pid_list}" if pid_list else "-C java -C jsvc"
    
    try:
        java_pid_list = subprocess.check_output(f"ps {ps_process_select_options} -o pid --no-headers", shell=True).decode().strip()
        if not java_pid_list:
            raise subprocess.CalledProcessError(1, "No Java process found!")

        java_pid_list = ",".join(java_pid_list.split())

        top_cmd_line = f"top -H -b -d {cpu_sample_interval} -n 2 -p {java_pid_list}"
        top_out = subprocess.check_output(top_cmd_line, shell=True, env={"HOME": tmp_store_dir}).decode()

        if store_dir:
            with open(f"{store_file_prefix}{update_round_num + 1}_top", 'w') as f:
                f.write(top_cmd_line + "\n" + top_out)

        # Parse the output to get thread ID and CPU usage from the second snapshot
        result_threads_top_info = []
        block_index = 0
        previous_line = None
        for line in top_out.splitlines():
            if previous_line and not line.strip():
                block_index += 1
            if block_index == 3 and line.strip() and line.split()[0].isdigit():
                result_threads_top_info.append((line.split()[0], line.split()[8]))  # thread ID and %CPU
            previous_line = line

        if not result_threads_top_info:
            die("No Java threads found in `top` output!")

        return sorted(result_threads_top_info, key=lambda x: float(x[1]), reverse=True)

    except subprocess.CalledProcessError:
        die("No Java process found!")

def __complete_pid_user_by_ps(threads):
    """Complete PID and user information using `ps`."""
    ps_process_select_options = f"-p {pid_list}" if pid_list else "-C java -C jsvc"
    ps_cmd_line = f"ps {ps_process_select_options} -wwLo 'pid,lwp,user' --no-headers"

    try:
        ps_out = subprocess.check_output(ps_cmd_line, shell=True).decode()

        if store_dir:
            with open(f"{store_file_prefix}{update_round_num + 1}_ps", 'w') as f:
                f.write(ps_cmd_line + "\n" + ps_out)

        results = []
        for thread_id, pcpu in threads:
            for line in ps_out.splitlines():
                pid, lwp, user = line.split()[:3]
                if lwp == thread_id:
                    results.append((pid, lwp, pcpu, user))
                    break

        return results

    except subprocess.CalledProcessError:
        die("No Java process found!")

def print_stack_of_threads(threads):
    """Print the stack trace of busy threads using `jstack`."""
    idx = 0
    for pid, thread_id, pcpu, user in threads:
        idx += 1
        thread_id_hex = format(int(thread_id), 'x')

        jstack_file = f"{store_file_prefix}{update_round_num + 1}_jstack_{pid}"
        if not os.path.isfile(jstack_file):
            jstack_cmd_line = [jstack_path, force, mix_native_frames, lock_info, pid]
            try:
                if user == WHOAMI:
                    with open(jstack_file, 'w') as f:
                        subprocess.run(jstack_cmd_line, stdout=f, check=True)
                elif os.geteuid() == 0:
                    with open(jstack_file, 'w') as f:
                        subprocess.run(['sudo', '-u', user] + jstack_cmd_line, stdout=f, check=True)
                else:
                    print(f"[{idx}] Fail to jstack busy({pcpu}%) thread({thread_id}/{thread_id_hex}) "
                          f"stack of java process({pid}) under user({user}).")
                    print(f"User of java process({user}) is not current user({WHOAMI}), need sudo to rerun:")
                    print(f"    sudo {' '.join(print_calling_command_line())}")
                    continue
            except subprocess.CalledProcessError:
                print(f"[{idx}] Failed to jstack busy({pcpu}%) thread({thread_id}/{thread_id_hex}) "
                      f"stack of java process({pid}) under user({user}).")
                if os.path.exists(jstack_file):
                    os.remove(jstack_file)
                continue

        print(f"[{idx}] Busy({pcpu}%) thread({thread_id}/{thread_id_hex}) stack of java process({pid}) under user({user}):")
        with open(jstack_file, 'r') as f:
            stack_trace = f.read()
            print(stack_trace)



def main():
    global update_round_num
    update_round_num = 0
    
    # Infinite loop if update_count <= 0, otherwise loop update_count times
    while update_count <= 0 or update_round_num < update_count:
        if update_round_num > 0:
            time.sleep(update_delay)
            normal_output()

        # Capture the current timestamp
        timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S.%f")

        # If append_file or store_dir is specified, print header info and write to files
        if append_file or store_dir:
            head_info(timestamp, update_round_num)
            if append_file:
                with open(append_file, 'a') as f:
                    f.write(f"{head_info(timestamp, update_round_num)}\n")
            if store_dir:
                with open(f"{store_file_prefix}{PROG}_log", 'a') as f:
                    f.write(f"{head_info(timestamp, update_round_num)}\n")

        # Print header info if update_count is not 1
        if update_count != 1:
            head_info(timestamp, update_round_num)

        # Find busy threads using ps or top depending on cpu_sample_interval
        if cpu_sample_interval == 0:
            busy_threads = find_busy_java_threads_by_ps()
        else:
            busy_threads = find_busy_java_threads_by_top()

        # Print the stack trace of the busy threads
        print_stack_of_threads(busy_threads)

        # Increment the update round number
        update_round_num += 1

if __name__ == "__main__":
    main()
