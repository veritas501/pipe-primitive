# pipe-primitive
An exploit primitive in linux kernel inspired by DirtyPipe (CVE-2022-0847).

</br>

前些日子，我像众多安全前辈那样对DirtyPipe（CVE-2022-0847）漏洞进行了学习和复现，深深感觉到这个洞的好用，这个洞始于一处内存的未初始化问题，终于对任意文件的修改，且中途不涉及KASLR的leak以及ROP、JOP等操作。因此无需绕过SMEP，SMAP等保护。

复现完后，我开始思考，DirtyPipe这个洞为什么好用？难道随着DirtyPipe的修复，它就会像昙花一现一样，对将来的漏洞利用不具备任何学习价值吗？

突然，我意识到，DirtyPipe所在结构体——`struct pipe_buffer`，似乎非常的耳熟。啊！它不正是那个用`GFP_KERNEL_ACCOUNT`flag分配的，位于slab-1k中，且包含一个ops字段常被我拿来用于做KASLR leak和RIP劫持的结构体吗？

随即我感到自己像个傻子，当时为什么要去修改pipe_buffer的ops做ROP？直接修改pipe_buffer的flags并配合splice不就直接做到任意文件写了嘛！这样就不用leak kaslr，也就不需要对不同内核版本做exploit适配，也不用管gadgets的事情，也不需要绕过SMEP，SMAP，KPTI等保护了。

我马上掏出了之前复现学习CVE-2021-22555及其他涉及pipe_buffer结构体的exploit，稍作修改，马上就能在未打补丁的不同版本内核中利用成功，下面是几个列子：

- https://github.com/veritas501/CVE-2021-22555-PipeVersion
- https://github.com/veritas501/CVE-2022-0185-PipeVersion
- https://github.com/veritas501/CVE-2022-25636-PipeVersion

由于在这之前并没有看到有人将其作为一种原语用于Linux kernel exploit中，所以本人就自作主张的将其称为“**Pipe Primitive**”吧 XD。

</br>

**总结一下**，就是需要获取到对Pipe结构体的修改能力，比如可以是slab-1k下的UAF，或是slab-1k下带偏移的（即不是按顺序的直接overflow）越界写。

这样，在kernel >= 5.8中，我们只需修改pipe_buffer中splice页的`flag |= PIPE_BUF_FLAG_CAN_MERGE`即可（有能力可以顺便把offset和len改成0，这样就能从文件的开头开始写）；在kernel < 5.8中，需要先leak一下pipe_buffer中的anon_pipe_ops，然后将splice页的的ops改为anon_pipe_ops（因为<5.8版本中能否merge是看ops的）（有能力依然可以顺便把offset和len改成0）。

</br>

至于你问这样的话容器逃逸怎么做？我想要做的应该和CVE-2019-5736差不多吧 XD。
