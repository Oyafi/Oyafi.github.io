---
title: 本地部署senseVoice
date: 2026-06-06 20:52:19
tags: 
---

## 本地部署senseVoice

最近在寻找日常实习，进行了大量的面试，每次面试完都需要将录音转文字然后交给大模型分析本次面试的表现情况和标准答案，之前使用讯飞听见进行录音转文字的操作，但是标准的档位一个月仅有五个小时的额度，下一个档位五十个小时，但是要接近两百块钱，有点遭不住，于是决定本地部署一个语言转文字的专用模型，了解到senseVoice效果还不错且开源，于是本地部署了一个，将我的操作路径分享给大家。

### 环境配置

首先下载项目

> git clone https://github.com/FunAudioLLM/SenseVoice.git

进入项目根目录安装依赖

> pip install -r requirements.txt
