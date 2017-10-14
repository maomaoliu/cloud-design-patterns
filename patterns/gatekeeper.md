# 看门人模式

通过使用专用主机实例来保护应用程序和服务，该主机实例充当客户端和应用程序或服务之间的代理，对请求进行验证和清理，并在它们之间传递请求和数据。这可以提供额外的安全层，并限制系统的攻击面。

### 背景和问题

应用程序通过接受和处理请求将其功能暴露给客户端。在云托管方案中，应用程序会公开客户端连接的端点，通常包括处理客户端请求的代码。该代码执行认证和验证，部分或全部请求处理，并且可能代表客户端访问存储和其它服务。
如果恶意用户能够破坏系统并获得对应用程序托管环境的访问权限，那么它所使用的安全机制（如凭据和存储密钥）以及访问的服务和数据将暴露。结果就是恶意用户可以无限制地访问敏感信息和其它服务。

## 解决方案

为了最小化客户端访问敏感信息和服务的风险，将公开端点的主机或任务与处理请求和访问存储的代码分离。可以通过使用与客户端交互的门面或专用任务来实现此目的，然后将请求（也许通过解耦接口）切换到将处理请求的主机或任务。下图概述了这种模式。
![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/gatekeeper-diagram.png)
看门人模式可用于简化保护存储，或者可以用作更全面的门面来保护应用程序的所有功能。重要的因素有：
* **受控验证**。看门人验证所有请求，并拒绝那些不符合验证要求的请求。
* **有限的风险和暴露**。看门人无法访问可信主机使用的凭据或密钥来获得存储和服务的访问。如果看门人暴露，攻击者将无法访问这些凭据或密钥。
* **适当的安全**。看门人以有限的特权模式运行，而其余应用程序以访问存储和服务所需的完全信任模式运行。如果看门人受到威胁，则无法直接访问应用程序服务或数据。

这种模式像典型的网络拓扑中的防火墙一样。允许看门人检查请求并做出关于是否将请求传递给执行所需任务的受信任主机（有时称为关键主人）的决定。该决定通常要求看门人在将请求内容传递给受信任的主机之前对请求内容进行验证和清理。

### 问题和注意事项

在决定如何实现此模式时，请考虑以下几点：
* 确保看门人的信任主机通过请求，仅暴露内部或受保护的端点，并仅连接到看门人。受信任的主机不应暴露任何外部端点或接口。
* 看门人必须以有限的特权模式运行。这通常意味着在单独的托管服务或虚拟机中运行看门人和可信主机。
* 看门人不应该执行与应用程序或服务相关的任何处理，或访问任何数据。它的功能纯粹是为了验证和清理请求。受信任的主机可能需要对请求执行其它验证，但核心验证应由看门人执行。
* 在看门人和受信任的主机之间或者可能的任务使用安全通信通道（HTTPS，SSL或TLS）。但是，某些托管环境不支持内部端点上的HTTPS。
* 将额外的层添加到应用程序以实现看门人模式，这可能会对性能有一些影响，因为需要额外的处理和网络通信。
* 看门人实例可能存在单点故障。为了尽量减少故障的影响，请考虑部署其它实例并使用自动伸缩机制来确保维护可用性的能力。

## 何时使用该模式

在以下场景适用于该模式：
* 处理敏感信息的应用程序，暴露必须高度保护免受恶意攻击的服务，或执行不应中断的关键任务操作。
* 有必要与主要任务分开执行请求验证的分布式应用程序，或集中此验证以简化维护和管理。

## 案例
在云托管场景中，可以通过分离应用程序中受信任的角色和服务来实现看门人角色或虚拟机的解耦。通过使用内部端点，队列或存储作为中间通信机制来执行此操作。下图展示了使用内部端点的方式。
![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/gatekeeper-endpoint.png)

## 相关模式
实现看门人模式时，[Valet Key模式](valet-key.md)也可能是相关的。在Gatekeeper和受信任的角色之间进行通信时，通过使用限制访问资源的权限的密钥或令牌来增强安全性是个好习惯。描述如何使用令牌或密钥提供客户端对特定资源或服务的受限直接访问。