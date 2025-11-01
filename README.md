# debug-collection
📘 文件说明
本文件旨在帮助实验室新入门的开发者在环境配置、代码调试以及日常系统使用过程中，能更快速地定位与解决问题。
内容涵盖开发中常见的报错、依赖冲突、编译失败等情况，所有记录均按照 “问题 → 原因 → 解决办法” 的格式整理，方便检索与借鉴。 
本文件既可作为环境搭建与调试的参考手册，也能为团队成员在处理类似问题时提供快速排查思路。
记录内容会随着经验积累持续更新，形成可复用的知识库。 
________________________________________
💡 使用建议
1.	查找问题前先检索：优先通过关键词搜索已有记录，避免重复排查。
2.	按格式追加新记录：遇到新问题时，请按照“问题 / 原因 / 解决办法”的结构补充，确保条理统一、便于他人理解。
3.	记录关键版本信息：每次问题描述中请尽量注明系统环境、Python 和库版本号，这对后续复现非常重要。
4.	验证后再更新：确认有效的解决方案再提交到文档中，保持内容质量和可参考性。
5.	问题类型不限，鼓励广泛记录：
可记录的内容不仅限于代码错误或依赖问题，还包括各种实际工作中遇到的技术困难，例如：
	特定专业软件的安装与配置问题
	云服务器、集群、远程环境连接及权限相关问题
	操作系统使用中遇到的故障、性能问题或安全设置（如 NVIDIA 驱动冲突、Linux 文件权限、Windows 环境变量）
	开发工具链问题（如 Git、Docker、Conda、VSCode 插件兼容性等）
	数据处理或格式转换过程中的异常（如 CSV 编码错误、JSON 解析失败）
	网络与依赖下载问题（如 SSL 认证、本地代理配置等）
只要是影响工作流程或新人入门的技术性问题，皆可记录，让文档成为全面的“经验索引库”。
🛠️ 维护与更新说明
本文件为持续更新的内部参考资料，旨在积累和传承日常调试与问题解决经验。
任何成员在遇到新问题或发现已有解决方案可优化时，均可直接在文档尾部追加新记录或修订原条目。
建议： 
•	定期（如每季度或每次大版本环境更新后）进行一次整体审查与内容整理；
•	为保证可追溯性，可在每条记录末尾注明记录人及日期；
•	若问题涉及多个系统或框架版本，请尽量注明适用范围或特殊依赖；
•	若发现重复或过时的内容，请在更新时保留历史备注，以免造成信息丢失。
通过共同维护，本文件将逐步沉淀为覆盖广泛、结构清晰、可复用性强的团队知识库，为新成员快速上手与老成员高效排错提供长期支持。 

 
1.	Debug记录 1：证书报错（pip 安装包时 SSL 认证失败）
问题：执行命令
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple lpips
证书报错：
Could not fetch URL https://pypi.tuna.tsinghua.edu.cn/simple/lpips/: There was a problem confirming the ssl certificate: [SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed (_ssl.c:748) – skipping

原因：
请求清华镜像源时 pip 在 SSL 证书验证阶段失败，通常是由于网络环境中的证书校验链不完整或 pip 版本兼容性问题导致

解决办法：在安装命令中添加 --trusted-host 参数，跳过 SSL 验证：
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple --trusted-host pypi.tuna.tsinghua.edu.cn lpips

2.	Debug记录 2：PyTorch 配置 EMD Loss 时版本不匹配
问题：使用 PyTorchEMD 时，出现 Torch 与 CUDA 版本不兼容。

原因：
原始 PyTorchEMD 代码基于较老版本的 PyTorch（<1.10），其中的接口（如 THC/THC.h、AT_CHECK、THCudaCheck 等）在新版本中已被替代或弃用。

解决办法：针对 cuda11.6 + torch1.13.1 环境，修改 cuda/emd_kernel.cu：
（参考github链接：https://github.com/daerduoCarey/PyTorchEMD）
对于cuda11.6+torch1.13.1，需要：
modified cuda/emd_kernel.cu as:
Comment #include <THC/THC.h>
Repalce all AT_CHECK with TORCH_CHECK
Replace all THCudaCheck with C10_CUDA_CHECK
Replace all CHECK_EQ with TORCH_CHECK_EQ

3.	Debug记录 3：H100 上编译 pointnet2_ops_lib 报错（不支持旧 GPU 架构）
问题：在 H100（CUDA 12.1）环境下执行编译报错：
在H100上安装pointnet2_ops_lib时由于cuda新版本与旧代码不一致导致的问题：
nvcc fatal   : Unsupported gpu architecture 'compute_37'
error: command '/usr/local/cuda-12.1/bin/nvcc' failed with exit code 1

原因：旧版本 setup.py 中固定了低版本 GPU 架构（如 compute_37），CUDA 12 及 H100 不再支持该架构参数。

解决办法 ：
在pointnet2_ops_lib/setup.py的line 19处修改
# os.environ["TORCH_CUDA_ARCH_LIST"] = "3.7+PTX;5.0;6.0;6.1;6.2;7.0;7.5"
os.environ["TORCH_CUDA_ARCH_LIST"] = "5.0;6.0;6.1;6.2;7.0;7.5;8.0;8.6;9.0"
	在同个文件的line 28处修改
	    ext_modules=[
        CUDAExtension(
            name="pointnet2_ops._ext",
            sources=_ext_sources,
            extra_compile_args={
                "cxx": ["-O3"],
                "nvcc": [
                    "-O3",
                    "-Xfatbin",
                    "-compress-all",
                    "--generate-code=arch=compute_90,code=sm_90",  
                    "-D_FORCE_INLINES",  # 增加内联强制以提高性能
                ],
            },
            include_dirs=[osp.join(this_dir, _ext_src_root, "include")],
        )
],

在pointnet2_ops_lib/pointnet2_ops/_ext-src/src/sampling_gpu.cu文件的前几行添加
#if defined(__CUDA_ARCH__) && __CUDA_ARCH__ >= 900  
#define CUDA_ARCH_90_SUPPORT
#endif

4.	Debug记录 4：安装 PyTorchEMD 时编译错误（找不到 THC/THC.h）
类似于记录2，这里记录另一种解决办法.
emdloss对应的github链接：https://github.com/daerduoCarey/PyTorchEMD
问题：运行python setup.py install时出错；
安装pytorch_emd时，运行python setup.py install时出现的错误：cuda/emd_kernel.cu:14:10: fatal error: THC/THC.h: No such file or directory
error: command '/usr/local/cuda-12.1/bin/nvcc' failed with exit code 1

原因：旧版 (<torch1.10) PyTorch 代码中使用的头文件 THC/THC.h 在新版中已移除。并且一些旧宏（如 CHECK_EQ, THCudaCheck）已被更新的 AT/C10 接口取代。

解决办法：
将emd_kernel.cu中
#include <THC/THC.h>
替换为
#include <torch/extension.h>
同时，按下列方式替换函数：
//  替换CHECK_EQ：例如，将CHECK_EQ(xyz2.size(0), b);改为AT_ASSERT(xyz2.size(0) == b);。注意，AT_ASSERT需要条件为布尔表达式。
//  替换THCudaCheck：例如，将THCudaCheck(cudaGetLastError());改为AT_CUDA_CHECK(cudaGetLastError());。AT_CUDA_CHECK会自动处理CUDA错误。
