# gradle

## gradle熟悉么，自动打包知道么？

## 如何加快 Gradle 的编译速度？
https://juejin.cn/post/6854573211548385294
https://developer.android.com/studio/build/optimize-your-build?hl=zh-cn#optimize
https://cloud.tencent.com/developer/article/1597526




## Gradle的Flavor能否配置sourceset？

## Gradle生命周期
[[Gradle构建生命周期]]
[[Gradle生命周期详解]]




# 编译插桩

## 谈谈你对AOP技术的理解？

- 基于 Gradle Transform API 创建 TransForm ，其执行时机在 class 被打包成 dex 之前
- 在 TransForm 中通过 javassist 或 asm 修改字节码
- 基于 Gradle Plugin API 自定义插件，应用自定义的 TransForm

## 说说你了解的编译插桩技术？

