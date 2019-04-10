title: 实验文档3：在kubernetes集群里集成Apollo配置中心
author: Stanley Wang
categories: Kubernetes容器云技术专题
date: 2019-1-18 20:12:56
---
# Apollo简介
Apollo（阿波罗）是携程框架部门研发的分布式配置中心，能够集中化管理应用不同环境、不同集群的配置，配置修改后能够实时推送到应用端，并且具备规范的权限、流程治理等特性，适用于微服务配置管理场景。

## 官方GitHub地址
[Apollo官方地址](https://github.com/ctripcorp/apollo)

## 基础架构
![apollo基础架构](/images/apollo.png "apollo基础架构")

## 简化模型
![apollo简化架构](/images/apollo-simple.png "apollo简化架构")

# 交付Apollo至Kubernetes集群
## 交付apollo-configservice
## 交付apollo-adminservice
## 交付apollo-portal

# 改造dubbo微服务接入apollo配置中心
## 改造dubbo-demo-service
## 改造dubbo-demo-web

# 实战通过配置中心维护多套环境项目
## TEST
### kubernetes配置
### apollo配置

## PRD
### kubernetes配置
### apollo配置

# 附：改造dubbo-demo-web为tomcat启动项目