---
title: ASP.NET Core MVC 中基于视图的授权
author: rick-anderson
description: 本文档演示如何在 ASP.NET Core Razor视图中注入和使用授权服务。
monikerRange: '>= aspnetcore-2.1'
ms.author: riande
ms.date: 11/08/2019
no-loc:
- Blazor
- Identity
- Let's Encrypt
- Razor
- SignalR
uid: security/authorization/views
ms.openlocfilehash: 1a3b4a92d270844fa99fc9fb0283226e94edb22d
ms.sourcegitcommit: 70e5f982c218db82aa54aa8b8d96b377cfc7283f
ms.translationtype: MT
ms.contentlocale: zh-CN
ms.lasthandoff: 05/04/2020
ms.locfileid: "82775617"
---
# <a name="view-based-authorization-in-aspnet-core-mvc"></a>ASP.NET Core MVC 中基于视图的授权

开发人员通常需要根据当前用户标识来显示、隐藏或修改 UI。 可以通过[依赖关系注入](xref:fundamentals/dependency-injection)访问 MVC 视图中的授权服务。 若要将授权服务注入Razor视图，请使用`@inject`指令：

```cshtml
@using Microsoft.AspNetCore.Authorization
@inject IAuthorizationService AuthorizationService
```

如果要在每个视图中设置授权服务，请`@inject`将指令放入*Views*目录的 *_ViewImports.* 有关详细信息，请参阅[视图中的依赖关系注入](xref:mvc/views/dependency-injection)。

使用注入的授权服务进行调用`AuthorizeAsync`的方法与在[基于资源的授权](xref:security/authorization/resourcebased#security-authorization-resource-based-imperative)期间进行检查的方式完全相同：

```cshtml
@if ((await AuthorizationService.AuthorizeAsync(User, "PolicyName")).Succeeded)
{
    <p>This paragraph is displayed because you fulfilled PolicyName.</p>
}
```

在某些情况下，资源将是您的视图模型。 调用`AuthorizeAsync`的方法与在[基于资源的授权](xref:security/authorization/resourcebased#security-authorization-resource-based-imperative)期间进行检查的方式完全相同：

```cshtml
@if ((await AuthorizationService.AuthorizeAsync(User, Model, Operations.Edit)).Succeeded)
{
    <p><a class="btn btn-default" role="button"
        href="@Url.Action("Edit", "Document", new { id = Model.Id })">Edit</a></p>
}
```

在前面的代码中，该模型将作为资源进行传递，策略评估应考虑到该资源。

> [!WARNING]
> 请不要依赖于应用的 UI 元素的切换可见性，作为唯一的授权检查。 隐藏 UI 元素可能无法完全阻止对其关联控制器操作的访问。 例如，请考虑前面代码片段中的按钮。 如果用户知道相对资源`Edit` URL 是 */Document/Edit/1*的，则用户可以调用操作方法。 出于此原因， `Edit`操作方法应执行其自己的授权检查。
