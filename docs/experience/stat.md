# stat系列syscall开发

## 开发经历
- 最早`basic`测例仅仅支持`x86_64`架构，初始的`starry-next`也只支持`x86_64`架构，所以最开始可以通过`basic`测例，但我们支持了四个架构的`stat`系列`syscall`之后`basic`测例无法通过，后更换了测例镜像使得`basic`测例支持四个架构后问题解决。