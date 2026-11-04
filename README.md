# wls-bench

2026 年下半年几乎每天都有大模型发布，不知道是从什么时候开始流行用画鹈鹕骑自行车来测试模型能力，可是这能测出来什么呢，难道软件工程能力仅凭画鹈鹕骑自行车就能测的出来吗，就算是许愿编程能力也测不出来啊。我就不会用 CSS 画鹈鹕骑自行车，但是哪个模型的工程能力能超过我。所以我认为每个人都应该以自己实际开发过程中遇到的需求和问题来测试模型，毕竟符合自己需求的模型才是最好的模型，而不是跑分高就是好模型。

该 Bench 目前包含以下四类测试：世界知识、许愿式编程、Bug 修复和多模态能力，题目后续可能会进行增删或修改。每道题的评分均为人类主观标准，纯主观，无客观标准。题目可能只测试了部分模型，因为我没钱用不起最新最好的模型，如果你有资源欢迎帮我测试，并将测试结果通过 PR 提交至本仓库。

## 世界知识/幻觉抑制

题目：

1. 比较 Kotlin 内联函数的死代码优化差异：[1-mmkv](knowledge/1-mmkv/question.md)

结果：

|模型|1-mmkv|
|----|------|
|人类|√|
|GPT 5.6 Sol xhigh|√|
|Gemini 3.1 Pro|×|
|DeepSeek V4 Flash|-|
|DeepSeek V4 Pro|×|
|DeepSeek V4.1 Flash 1017|√|
|GLM 5.2|×|
|GLM 5.3|×|
|GLM 5.3 Flash|部分正确|
|Qwen 3.8 Max|×|
|Qwen 3.8 Flash|×|
|Kimi K3|×|
|MiniMax M3.1 Flash|×|
|XiaoMi Mimo 2.6 Pro|×|
|HY 4|部分正确|
|Step 5|×|

## 许愿式编程

题目：

1. 佩丽卡骑自行车：[1-plkbike](vibecoding/1-plkbike/question.md)
2. 咖啡店点餐系统：[2-musiccafe](vibecoding/2-musiccafe/question.md)

结果：

|模型|1-plkbike|2-musiccafe|
|----|---------|-----------|
|人类|-|-(我要是想做这么无聊的工作我还许什么愿了)|
|DeepSeek V4.1 Flash 1017|2.5/5.0(第一次脸画错了，且没有理解黎博利的特征，但是至少画的像个人)|-|
|HY 4 preview|1.0/5.0(至少能动)|-|
|GLM 5.3 Flash|-|3.0/5.0(无许愿编程能力，只会机械地实现需求，但实现得较好，问题通过提示都能改正)|

## Bug 修复

题目：

1. CRP 播放 RTP 视频流图像抖动问题：[1-crp](bugfix/1-crp/question.md)

结果：

|模型|1-crp|
|----|-----|
|人类|5.0/5.0|
|GPT 5.6 Sol max|3.5/5.0(找到了根因，但改了大量无关的代码)|
|DeepSeek V4.1 Flash 0910|0.0/5.0(提示好几轮仍无法找出根因，0910之前的DeepSeek Flash/Pro也是)|
|DeepSeek V4.1 Flash 1017|4.0/5.0(和GPT 5.6 Sol max相同的侵入式修改，但是没改无关代码)|
|GLM 5.3|3.0/5.0(大量提示下才找到根因，且有对live555的侵入式修改)|
|GLM 5.3 Flash|4.0/5.0(有对live555的侵入式修改)|
|Qwen 3.8 Max Preview|3.0/5.0(提示几轮后和GPT 5.6 Sol max相同的侵入式修改，改了无关代码，且有雷霆大思考)|
|Kimi K3|4.5/5.0(核心改动与我完全一致，但也改了无关的代码)|
|MiniMax M3.1 Flash|0.0/5.0(区)|
|XiaoMi Mimo 2.6 Pro|3.0/5.0(侵入式修改+无关修改，且雷霆大思考，非常慢)|
|HY 4 preview|4.0/5.0(找到了根因，但是有对live555的侵入式修改)|
|Step 5|4.5/5.0(核心改动与我完全一致，但是有对live555的侵入式修改)|

## 多模态能力

题目：

1. 识别衣服上的 Base64 字符串：[1-peaceforall](multimodal/1-peaceforall/question.md)
2. 学校模型分割：[2-modelseg](multimodal/2-modelseg/question.md)

结果：

|模型|1-peaceforall|2-modelseg|
|----|-------------|----------|
|人类|×(OCR准确率太低，我又懒得一个一个字敲)|×(我还没学会blender呢)|
|GPT 5.6 Sol xhigh|√|-|
|DeepSeek V4.1 Flash 1017|-|4.0/5.0(指出分割错误的地方后能修复成功，多轮修复后完成任务)|
|GLM 5.3 Flash|-|1.0/5.0(经过几十轮修改后仍无法正确分割，且多次出现模型、贴图错乱的情况)|
