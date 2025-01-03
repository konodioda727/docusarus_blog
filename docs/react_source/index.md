---
sidebar_position: 1
---

# 从头开始梳理`React`

## 前言

写这篇文章主要出于两个目的：

- 自己对于`React`的了解不够深入，希望通过写一篇文章来梳理自己的所学，以免日后忘记
- 目前网络上主流的源码解释都是从`React`的四大结构开始，然后深入解释每个结构。这样的结构固然很不错，但对于普通开发者来说，比较晦涩且很多时候会不知所云。因此我想尝试从开发者能见的角度厘清`React`的执行机制
- 网络上的资料（无论英文还是中文）同质化严重，没有几篇能够讲明白`React`这一块，并且`React`的架构也在更新，需要新文章来解释新架构

基于以上原因,本人查阅了很久的源码,最终产出了这一系列文章

本篇文章为开篇,介绍 react 的由来以及其中知名架构的起源,希望能帮助大家更好地理解 react

## React 简介
React 的原理以及思想其实很简单,在这篇文章中,我将其分为: 
- 为什么会有 React 
- React 干了什么
- 为什么会有 VDOM
- 为什么会有 Fiber
- 为什么会有 Scheduler
几个部分展开


以下是版块传送门:
- [React fiber](./fiber/renderPhase/index.md)
- [React Hooks](./hooks/index.md)
- [React algo](./algo/React-Reconcile.md)