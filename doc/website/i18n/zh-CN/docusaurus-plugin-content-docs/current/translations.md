---
id: translations
title: Translations
---

## 简介

为了让Ent框架更便于非英语母语者使用，我们启动了文档翻译计划。目标是将本站所有文档提供中文和日语版本。为简化翻译、审核及部署流程，我们已通过[Crowdin](https://crowdin.com)平台实现本地化集成。

## 参与贡献

1. 在我们的[Crowdin项目](https://crwd.in/ent)注册成为译者
2. 完成注册登录后，访问[项目主页](https://crowdin.com/project/ent)
3. 选择您要翻译的目标语言
4. 选择待翻译文档：
   * 所有文档位于`md/`目录下
   * 博客文章位于`website/blog`
   * 网站UI组件在`website/i18n`目录
5. 有关Crowdin编辑器的使用说明，请参阅[在线编辑器文档](https://support.crowdin.com/online-editor/)
6. 您的翻译建议经校对人员审核通过后，将随下次网站更新发布

## 申请成为校对员

若希望长期参与特定语言的翻译质量把控，请通过Slack频道联系我们申请成为校对员

## 重要准则

- 文档首行的`id: <文档ID>`标记禁止翻译，修改会导致网站构建失败
- 网站构建对HTML标签闭合敏感，翻译中若需使用HTML必须确保标签正确闭合或使用自闭合形式（如换行符应写作`<br/>`而非`<br>`）