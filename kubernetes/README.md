# kubernetes分析与学习
## 主要内容
本部分的内容主要是对kubernetes的分析，主要的学习路径为：先从整体架构开始，在了解整体架构以及工作流程的基础上，再对各个关键组件从顶向下的分析。对于各个组件来说，也是先从这个主要的主要功能开始，分析它的工作流程，再将它的工作流程拆分成各个细节，最后攻读其源代码进行分析。
## 本章构成
* 前导知识
* 各个关键组件的分析以及源码的阅读

## 目录
* [前导知识](overview)
* [核心组件](core_components)
	* [kube-scheduler](kube-scheduler)
	* [kube-api-server](kube-api-server)
	* [kube-controller-manager](kube-controller-manager)
	* [kube-proxy](kube-proxy)
	* [kubelet](kubelet)

## 预备知识
* 建议先了解一下什么是容器化，可以去了解一下docker
* 由于kubernetes是golang开发的，建议先去了解一下golang语言的语法

## FAQ
此部分用于大家问题的记录与讨论，在阅读的过程中有任务问题或新的发现都可以记录在此，大家可以在此畅所欲言。比如：

1. 我最近发现一个不错的docker构建工具：kaniko，它可以不依赖docker daemon