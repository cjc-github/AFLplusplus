
# 尝试绕过root来运行fuzz

案例：

```bash
git clone https://github.com/fuzzstati0n/fuzzgoat.git
cd fuzzgoat
gcc -o fuzzgoat -I. main.c fuzzgoat.c -lm

./afl-fuzz -i ./fuzzgoat/in/ -o ./fuzzgoat/out -n -- ./fuzzgoat/fuzzgoat @@
```

## 问题1：echo core >/proc/sys/kernel/core_pattern出现

```bash
[*] Checking core_pattern...

[-] Hmm, your system is configured to send core dump notifications to an
    external utility. This will cause issues: there will be an extended delay
    between stumbling upon a crash and having this information relayed to the
    fuzzer via the standard waitpid() API.
    If you're just testing, set 'AFL_I_DONT_CARE_ABOUT_MISSING_CRASHES=1'.

    To avoid having crashes misinterpreted as timeouts, please log in as root
    and temporarily modify /proc/sys/kernel/core_pattern, like so:

    echo core >/proc/sys/kernel/core_pattern

[-] PROGRAM ABORT : Pipe at the beginning of 'core_pattern'
         Location : check_crash_handling(), src/afl-fuzz-init.c:2399
```

解决方案：

```bash
export AFL_I_DONT_CARE_ABOUT_MISSING_CRASHES=1
./afl-fuzz -i ./fuzzgoat/in/ -o ./fuzzgoat/out -- ./fuzzgoat/fuzzgoat @@
```

## 问题2：cd /sys/devices/system/cpu; echo performance | tee cpu*/cpufreq/scaling_governor出现

```bash
Checking CPU scaling governor...

[-] Whoops, your system uses on-demand CPU frequency scaling, adjusted
    between 878 and 1357 MHz. Unfortunately, the scaling algorithm in the
    kernel is imperfect and can miss the short-lived processes spawned by
    afl-fuzz. To keep things moving, run these commands as root:

    cd /sys/devices/system/cpu
    echo performance | tee cpu*/cpufreq/scaling_governor

    You can later go back to the original state by replacing 'performance'
    with 'ondemand' or 'powersave'. If you don't want to change the settings,
    set AFL_SKIP_CPUFREQ to make afl-fuzz skip this check - but expect some
    performance drop.

[-] PROGRAM ABORT : Suboptimal CPU scaling governor
         Location : check_cpu_governor(), /home/test/tcl/AFLpp-Android-Greybox/fuzzers/aflpp/AFLplusplus-4.31c/src/afl-fuzz-init.c:2614
```

解决方案：

```bash
export AFL_SKIP_CPUFREQ=1

```

## 问题3：afl-fuzz 报错 Suboptimal CPU scaling governor

```bash
[*] Creating hard links for all input files...
[+] Loaded a total of 4 seeds.

[-]  SYSTEM ERROR : shmget() failed, try running afl-system-config
    Stop location : afl_shm_init(), /home/test/tcl/AFLpp-Android-Greybox/fuzzers/aflpp/AFLplusplus-4.31c/src/afl-sharedmem.c:284
       OS message : Permission denied
```

解决方案：

```bash
重新编译
make clean
# 似乎不行
USE_MMAP=1 make

# 或者使用 TEST_MMAP
TEST_MMAP=1 make

AFL_NO_X86=1 TEST_MMAP=1 CC=/home/test/tcl/AFLpp-Android-Greybox/fuzzers/aflpp/android-ndk-r25c/toolchains/llvm/prebuilt/linux-x86_64/bin/armv7a-linux-androideabi26-clang make
```

验证：

```bash
# 确认使用了 POSIX 共享内存
./afl-fuzz --version

# 检查是否还有 shmget 调用
nm ./afl-fuzz | grep shmget
# 应该没有输出（如果完全移除了 shmget）

# 或者用 strings
strings ./afl-fuzz | grep shmget
# 如果看到 shmget，可能是在其他模块中
```