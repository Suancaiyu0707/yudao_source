title: Spring-Cloud-Gateway 源码解析 —— 路由（1.1）之 RouteDefinitionLocator 一览
date: 2020-01-10
tags:
categories: Spring-Cloud-Gateway
permalink: Spring-Cloud-Gateway/route-definition-locator-intro

---

# 1. 概述

本文主要对 **路由定义定位器 RouteDefinitionLocator 做整体的认识**。

在 [《Spring-Cloud-Gateway 源码解析 —— 网关初始化》](http://www.iocoder.cn/Spring-Cloud-Gateway/init/?self) 中，我们看到路由相关的组件 RouteDefinitionLocator / RouteLocator 的初始化。涉及到的类比较多，我们用下图重新梳理下 ：

![](http://www.iocoder.cn/images/Spring-Cloud-Gateway/2020_01_10/01.png)

* RouteDefinitionLocator 负责读取路由配置( `org.springframework.cloud.gateway.route.RouteDefinition` ) 。从上图中我们可以看到，RouteDefinitionLocator **接口**有四种实现 ：
    * PropertiesRouteDefinitionLocator ，从**配置文件**( 例如，YML / Properties 等 ) 读取。在 [TODO 【3008】]() 详细解析。
    * RouteDefinitionRepository ，从**存储器**( 例如，内存 / Redis / MySQL 等 )读取。在 [TODO 【3016】]() 详细解析。
    * DiscoveryClientRouteDefinitionLocator ，从**注册中心**( 例如，Eureka / Consul / Zookeeper / Etcd 等 )读取。在 [TODO 【3010】]() 详细解析。
    * CompositeRouteDefinitionLocator ，组合**多种** RouteDefinitionLocator 的实现，为 RouteDefinitionRouteLocator 提供**统一**入口。在 [本文](#) 详细解析。
    * 另外，CachingRouteDefinitionLocator 也是 RouteDefinitionLocator 的实现类，已经被 CachingRouteLocator 取代。
* RouteLocator 可以直接自定义路由( `org.springframework.cloud.gateway.route.Route` ) ，也

