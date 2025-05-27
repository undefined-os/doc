# misc

- 协助编写`Starry-Tutorial-Book`(https://azure-stars.github.io/Starry-Tutorial-Book)，已经提交`PR`并合并到上游仓库。

- 协助编写`OS大赛赛题协作文档`(https://github.com/oscomp/oskernel-testsuits-cooperation)，已经提交`PR`并合并到上游仓库。

- 无效测例：在调试测例`fpclassify_invalid_ld80`的过程中，发现本地`wsl`运行此测例行为和输出结果与在实验框架下相同，但流水线测评为`fail`。经过沟通和确认发现`fpclassify_invalid_ld80`的测例代码在`x86_64`下编译工具链会出现问题，后联系了编写测评流水线的同学进行修改，`ban`掉了这个测例。

- 测评流水线逻辑问题：在评测脚本中通过统计正确输出次数是否为4(对应`riscv64`/`aarch64`/`loongarch64`/`x86_64`四个架构)来判定测例是否通过。但如果两个测例有相同的正确输出，会导致正确输出统计量翻倍，导致错误，联系了维护测评流水线的同学进行修改，问题解决。