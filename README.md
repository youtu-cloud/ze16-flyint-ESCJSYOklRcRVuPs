


---


　　大家好，我是痞子衡，是正经搞技术的痞子。今天痞子衡给大家分享的是**i.MXRT1170 XECC开启及Data Swap功能对于外部RAM的访问性能影响**。


　　文接上篇 [《i.MXRT1170 XECC功能特点及其保护串行NOR Flash和SDRAM之道》](https://github.com)，这篇文章里痞子衡给大家介绍了 XECC 原理及在其使能下操作 NOR Flash 步骤（尤其涉及对 Flash 的 AHB 方式写），但文章里并没有涉及性能方面的评估。我们知道 RT1170 上内部 FlexRAM ECC 模块使能后对 TCM 访问性能几乎无影响，那么 XECC 使能后对于挂在 FlexSPI/SEMC 接口上的外部 PSRAM/SDRAM 访问性能是否有影响呢？今天我们就来聊聊这个话题：



> * Note：本文以 MIMXRT1170\-EVKB (Rev.B) 板卡上挂在 SEMC 接口的 16bit SDRAM \- W9825G6KH\-5I 读写测试为例，PSRAM 测试过程类似。


### 一、XECC功能测试


　　测试 XECC 对于 SDRAM 访问保护功能我们可以直接使用如下两个官方例程，其中 xecc\_single\_error 示例了单 bit 纠错（4bits数据单元而言），xecc\_multi\_error 示例了双 bit 报错（4bits数据单元而言），这两个例程都借助了 XECC 本身的 Error injection 特性人为制造数据 bit 错误来做测试（可以指定 32bits 数据块中任意位置和个数的 bit 出错）。



```
\SDK_2_16_000_MIMXRT1170-EVKB\boards\evkbmimxrt1170\driver_examples\xecc\semc\xecc_single_error\cm7  
\SDK_2_16_000_MIMXRT1170-EVKB\boards\evkbmimxrt1170\driver_examples\xecc\semc\xecc_multi_error\cm7  

```

　　痞子衡简单整合了上述两个例程代码到一个工程里，这样可以同时测单/双/多 bit 错误情况，其中主要代码摘录如下。此外为了方便观察不同的错误注入导致的结果，我们将待写入值 sdram\_writeBuffer\[0] 设为 0x00000000，这样发生无法纠错情况时读回的数据 sdram\_readBuffer\[0] 就应该等于错误注入值 errorData。



```
#include "fsl_xecc.h"
volatile uint32_t sdram_writeBuffer[0x1000];
volatile uint32_t sdram_readBuffer[0x1000];
int main(void)
{
    // 系统与 SDRAM 初始化代码省略...
    // 初始化 XECC_SEMC 模块，设置 SDRAM [0x80000000, 0x8007FFFF] 为 ECC 使能区域，其中前 256KB 是用户数据访问空间
    XECC_Deinit(XECC_SEMC);
    xecc_config_t config;
    XECC_GetDefaultConfig(&config);
    config.enableXECC     = true;
    config.enableWriteECC = true;
    config.enableReadECC  = true;
    //config.enableSwap = true;
    config.Region0BaseAddress = 0x80000000U;
    config.Region0EndAddress  = 0x80080000U;  // 256KB * 2
    XECC_Init(XECC_SEMC, &config);
    (void)EnableIRQ(XECC_SEMC_INT_IRQn);
    (void)EnableIRQ(XECC_SEMC_FATAL_INT_IRQn);
    SCB->SHCSR |= SCB_SHCSR_BUSFAULTENA_Msk;
    XECC_EnableInterrupts(XECC_SEMC, kXECC_AllInterruptsEnable);

    // 对 32bits 数据块进行错误注入（这里设定得是错误 bit 位置）
    uint32_t errorData = 0x00000001;
    XECC_ErrorInjection(XECC_SEMC, errorData, 0);
    sdram_writeBuffer[0] = 0x00000000U;

    // AHB 方式写数据进 SDRAM
    *(uint32_t *)0x80000000U = sdram_writeBuffer[0];
    // 关闭 DCache 代码省略...
    // AHB 方式从 SDRAM 读回数据
    sdram_readBuffer[0] = *(uint32_t *)0x80000000U;
    while ((!s_xecc_single_error) && (!s_xecc_multi_error))
    {
    }
    // 代码省略...
}

```

　　在放测试结果之前，我们先回顾一下 XECC 错误检测机制。在默认不开启 Data Swap 特性情况下，对于 32bits 数据块，XECC 可以纠正其中发生的 8bits 错误，但前提是按序分割开的每 4bits 数据单元仅能有 1bit 错误（即分散 bit 错误）。如果这 4bits 数据单元里有 2bit 错误，那 XECC 会检测出位置并报告；如果有 3/4bit 错误，那已经超出 XECC 处理能力，结果不可预期了。


　　但如果现场实际环境发生连续 bit 错误概率高于分散 bit 错误，这时候可以考虑开启 XECC Data Swap 功能，这时候 XECC 处理能力变成可以纠正连续 bit 错误，但对于分散 bit 错误就无法处理了（如下图所示，实际上 Swap 是将图左边 32bits 原数据打乱再重新组合成图右边 32bits 新数据）。


![](https://raw.githubusercontent.com/JayHeng/pzhmcu-picture/master/cnblogs/i.MXRT1170_XECC_data_swap.PNG)


　　根据上面的理论，我们现在再来看测试结果，那就非常合理了，有些情况下开启 Data Swap 增强了纠检错能力，有些情况下开启 Data Swap 却减弱了原来的纠检错能力。（注意，Error injection 设定的 errorData 是针对 swap 之后的数据 bit 序而言）



> * Note：单 32bits 数据块写读测试时，改动 L1 D\-Cache 操作代码位置（前移到 main 开始），会影响测试结果，这里留一个伏笔，以后具体分析。




| config.enableSwap | errorData | sdram\_writeBuffer\[0] | sdram\_readBuffer\[0] | 测试结果 |
| --- | --- | --- | --- | --- |
| false/true | 0x00000001 | 0x00000000 | 0x00000000 | bit0错误被纠正 |
| false/true | 0x11111111 | 0x00000000 | 0x00000000 | bit0,4,8,12,16,20,24,28错误被纠正 |
| false | 0x08040201 | 0x00000000 | 0x00000000 | bit0,9,18,27错误被纠正 |
| ture | 0x08040201 | 0x00000000 | 0x00000000 | （原性能增强）bit0,2,17,19错误被纠正 |
| false | 0x02040801 | 0x00000000 | 0x00000000 | bit0,11,18,25错误被纠正 |
| ture | 0x02040801 | 0x00000000 | 0x00000000 | （原性能增强）bit0,1,2,3错误被纠正 |
| false | 0x00000003 | 0x00000000 | 0x00000003 | bit0,1错误被检测 |
| true | 0x00000003 | 0x00000000 | 0x00000201 | （原性能减弱）bit0,9错误被检测 |
| false | 0x00000007 | 0x00000000 | 0x00000007 | 触发单bit纠错中断，出错bit0,1,2位置识别不准 |
| true | 0x00000007 | 0x00000000 | 0x00040201 | 触发单bit纠错中断，（原性能减弱）出错bit0,9,18位置识别不准 |
| false | 0x0000000F | 0x00000000 | 0x0000000F | 触发双bit检错中断，出错bit0,1,2,3位置识别不准 |
| true | 0x0000000F | 0x00000000 | 0x08040201 | 触发双bit检错中断，（原性能减弱）出错bit0,9,18,27位置识别不准 |


### 二、XECC性能测试


　　现在我们简单测试一下内核对 SDRAM 读写性能是否受 XECC 影响，就在 \\semc\\xecc\_multi\_error 例程基础之上，设计了如下代码，便于测试不同的数据块大小（比如 64KB，128KB，256KB），而且可以一次性对比没有 XECC 保护，加 XECC 保护，以及开启 XECC Data Swap 三种情况（中途需要 Deinit 再 Init XECC 模块）。



```
uint32_t readErrorCnt = 0;
uint32_t get_sdram_rw_block_time(uint32_t start, uint32_t size)
{
    uint64_t tickStart = life_timer_clock();
    readErrorCnt = 0;
    for (uint32_t idx = 0; idx < size; idx += 4)
    {
        *((uint32_t*)(start + idx)) = idx;
    }
    for (uint32_t idx = 0; idx < size; idx += 4)
    {
        uint32_t temp = *((uint32_t*)(start + idx));
        if (temp != idx)
        {
            readErrorCnt++;
        }
    }
    uint64_t tickEnd = life_timer_clock();
    return ((tickEnd - tickStart) / (CLOCK_GetRootClockFreq(kCLOCK_Root_Bus) / 1000000));
}

void test_xecc_sdram_perf(uint32_t size)
{
    xecc_config_t config;
    XECC_GetDefaultConfig(&config);
    config.enableXECC     = true;
    config.enableWriteECC = true;
    config.enableReadECC  = true;
    config.Region0BaseAddress = 0x80040000;
    config.Region0EndAddress  = 0x800C0000;
    XECC_Deinit(EXAMPLE_XECC);
    // 写读 SDRAM 非 XECC 保护区域
    uint32_t normalTimeInUs = get_sdram_rw_block_time(0x80000000, size);
    // 写读 SDRAM XECC 保护区域（不使能 Data Swap）
    config.enableSwap = false;
    XECC_Init(XECC_SEMC, &config);
    uint32_t xeccNoSwapTimeInUs = get_sdram_rw_block_time(0x80040000, size);
    // 写读 SDRAM XECC 保护区域（使能 Data Swap）
    XECC_Deinit(XECC_SEMC);
    config.enableSwap = true;
    XECC_Init(XECC_SEMC, &config);
    uint32_t xeccSwapTimeInUs = get_sdram_rw_block_time(0x80040000, size);

    PRINTF("---------------------------------------\r\n");
    PRINTF("Write/Read/Compare data size: %d\r\n", size);
    PRINTF("Write/Read/Compare time in SDRAM region XECC disable : %d us\r\n", normalTimeInUs);
    PRINTF("Write/Read/Compare time in SDRAM region XECC enable without data swap : %d us\r\n", xeccNoSwapTimeInUs);
    PRINTF("Write/Read/Compare time in SDRAM region XECC enable with data swap : %d us\r\n", xeccSwapTimeInUs);
}

```

　　get\_sdram\_rw\_block\_time() 函数里实际上也统计了数据出错情况，可用于辅助判断写读是否正常，在恩智浦开发板测试环境下应该无 ECC 错误，所以这里结果就略去不表了。


　　我们现在来看 800MHz 内核主频下对 200MHz SDRAM 访问性能情况（代码跑在 ITCM 上），为了结果可靠，本次测试内核频率没有设到最高。根据代码打印结果，我们能得到如下结论：



> * 结论1：在无 XECC 保护情况下，开启 D\-Cache 能将 SDRAM 读写整体访问速度提升到 4 倍。
> * 结论2：XECC 使能情况下，是否开启 Data Swap 功能几乎不带来性能变化（在 XECC 外设里 Swap 操作一拍时钟就能完成，时延可以忽略）。
> * 结论3：XECC 使能情况下，会降低 SDRAM 读写整体访问性能，降幅在 28% （开启 D\-Cache） 或 11%（不开启 D\-Cache）。
> * 结论4：XECC 使能情况下，对于读写访问同样大小数据块，是否开启 D\-Cache，不影响 XECC 校验处理的时间（即 D\-Cache 不加速 XECC）。


![](https://raw.githubusercontent.com/JayHeng/pzhmcu-picture/master/cnblogs/i.MXRT1170_XECC_perf_test.PNG)


　　至此，i.MXRT1170 XECC开启及Data Swap功能对于外部RAM的访问性能影响痞子衡便介绍完毕了，掌声在哪里\~\~\~


### 欢迎订阅


文章会同时发布到我的 [博客园主页](https://github.com)、[CSDN主页](https://github.com):[veee加速器官网](https://youhaochi.com)、[知乎主页](https://github.com)、[微信公众号](https://github.com) 平台上。


微信搜索"**痞子衡嵌入式**"或者扫描下面二维码，就可以在手机上第一时间看了哦。


![](https://raw.githubusercontent.com/JayHeng/pzhmcu-picture/master/github/pzhMcu_qrcode_258x258.jpg)


