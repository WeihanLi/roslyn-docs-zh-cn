编译器查询
-------------

本页面记录了编译器团队用于管理 GitHub issues 的查询

## 常规

- [分类](https://github.com/dotnet/roslyn/issues?utf8=✓&q=is%3Aopen+-label%3AArea-IDE+-label%3Aarea-performance+-label%3Aarea-interactive+-label%3A"Area-SDK+and+Samples"+-label%3AArea-external+-label%3Aarea-infrastructure+-label%3A"Area-Language+Design"+-label%3Aarea-Analyzers+is%3Aissue+-label%3Abug+-label%3Atest+-label%3Astory+-label%3Adocumentation+-label%3A"feature+request"++-label%3Adocumentation+-label%3Aquestion+-milestone%3Abacklog+-label%3Ainvestigating+-label%3A"area-project+system"++-label%3A"Investigation+Required"++-label%3Adiscussion+-label%3A"Need+More+Info"+-label%3Adisccussion+-label%3A"Concept-Design+Debt")
- [拉取请求](https://github.com/dotnet/roslyn/pulls?utf8=%E2%9C%93&q=is%3Aopen+is%3Apr+-label%3A%22PR+For+Personal+Review+Only%22+label%3Aarea-compilers++)

## 特定版本
- [16.4](https://github.com/dotnet/roslyn/issues?utf8=✓&q=is%3Aopen+is%3Aissue+milestone%3A16.4+label%3AArea-Compilers+-label%3Adocumentation)
- [16.5](https://github.com/dotnet/roslyn/issues?utf8=✓&q=is%3Aopen+is%3Aissue+milestone%3A16.5+label%3AArea-Compilers+-label%3Adocumentation)
- [16.6](https://github.com/dotnet/roslyn/issues?utf8=✓&q=is%3Aopen+is%3Aissue+milestone%3A16.6+label%3AArea-Compilers+-label%3Adocumentation)

## 分阶段分类
- [阶段 1 - 未分配区域的 issues](https://github.com/dotnet/roslyn/issues?utf8=%E2%9C%93&q=is%3Aopen%20is%3Aissue%20-label%3A%22Area-Dynamic%20Analysis%22%20-label%3A%22Area-Project%20System%22%20%20-label%3AArea-Performance%20-label%3AArea-Analyzers%20-label%3AArea-Compilers%20-label%3AArea-Debugging%20-label%3A%22Area-Design%20Notes%22%20-label%3AArea-External%20-label%3AArea-IDE%20-label%3AArea-Infrastructure%20-label%3AArea-Interactive%20-label%3A%22Area-Language%20Design%22%20-label%3A%22Area-SDK%20and%20Samples%22%20-label%3A%22Sprint%20Summary%22%20)
- [阶段 2 - 尚未被"分类"为 Bug/Test/Story/Documentation/Feature/Question/Investigating/Investigation Required/Need More Info/Blocked/Tenet-Performance/Code Gen Quality/Concept-Design Debt/Discussion 的编译器 issues](https://github.com/dotnet/roslyn/issues?utf8=✓&q=is%3Aopen+is%3Aissue+label%3AArea-Compilers+-label%3ABug+-label%3ATest+-label%3AStory+-label%3ADocumentation+-label%3A"Feature+Request"+-label%3AQuestion+-label%3A"Investigation+Required"+-label%3A"Need+More+Info"+-label%3ABlocked+-label%3AInvestigating+-label%3ATenet-Performance+-label%3A"Code+Gen+Quality"+-label%3A"Concept-Design+Debt"+-label%3ADiscussion+)
   - [2a 除 Nullable 以外](https://github.com/dotnet/roslyn/issues?utf8=✓&q=is%3Aopen+is%3Aissue+label%3AArea-Compilers+-label%3ABug+-label%3ATest+-label%3AStory+-label%3ADocumentation+-label%3A"Feature+Request"+-label%3AQuestion+-label%3A"Investigation+Required"+-label%3A"Need+More+Info"+-label%3ABlocked+-label%3AInvestigating+-label%3ATenet-Performance+-label%3A"Code+Gen+Quality"+-label%3A"Concept-Design+Debt"+-label%3ADiscussion++-label%3A%22New+Language+Feature+-+Nullable+Reference+Types%22)
   - [2b 仅 Nullable](https://github.com/dotnet/roslyn/issues?utf8=✓&q=is%3Aopen+is%3Aissue+label%3AArea-Compilers+-label%3ABug+-label%3ATest+-label%3AStory+-label%3ADocumentation+-label%3A"Feature+Request"+-label%3AQuestion+-label%3A"Investigation+Required"+-label%3A"Need+More+Info"+-label%3ABlocked+-label%3AInvestigating+-label%3ATenet-Performance+-label%3A"Code+Gen+Quality"+-label%3A"Concept-Design+Debt"+-label%3ADiscussion+label%3A%22New+Language+Feature+-+Nullable+Reference+Types%22)
- [阶段 3 - 尚未分配版本的编译器 issues](https://github.com/dotnet/roslyn/issues?q=is%3Aopen+is%3Aissue+label%3AArea-Compilers+no%3Amilestone)
- [阶段 4 - 16.5 版本中尚未分配工程师的编译器 issues](https://github.com/dotnet/roslyn/issues?q=is%3Aopen+is%3Aissue+label%3AArea-Compilers+milestone%3A16.5+no%3Aassignee)
- [可能应被关闭或移至语言规范仓库的语言设计 issues](https://github.com/dotnet/roslyn/issues?utf8=%E2%9C%93&q=is%3Aopen+is%3Aissue+label%3A%22Area-Language+Design%22+-label%3AArea-Compilers)
- [最久远的"需要更多信息"编译器 issues。如果没有提供信息则关闭](https://github.com/dotnet/roslyn/issues?q=is%3Aopen+is%3Aissue+label%3AArea-Compilers+label%3A%22Need+More+Info%22+sort%3Acreated-asc)
- [最久远的被阻塞编译器 issues](https://github.com/dotnet/roslyn/issues?q=is%3Aopen+is%3Aissue+label%3AArea-Compilers+sort%3Acreated-asc+label%3ABlocked)
- [最久远的编译器问题 - 请回答并关闭](https://github.com/dotnet/roslyn/issues?q=is%3Aopen+is%3Aissue+label%3AArea-Compilers+sort%3Acreated-asc+label%3AQuestion)

## 已分类 Issues
- [Bug](https://github.com/dotnet/roslyn/issues?q=is%3Aopen+is%3Aissue+label%3AArea-Compilers+label%3ABug)
- [Tenet-Performance](https://github.com/dotnet/roslyn/issues?q=is%3Aopen+is%3Aissue+label%3AArea-Compilers+label%3ATenet-Performance)
- [Code Gen Quality](https://github.com/dotnet/roslyn/issues?q=is%3Aopen+is%3Aissue+label%3AArea-Compilers+label%3A%22Code+Gen+Quality%22)
- [Investigation Required](https://github.com/dotnet/roslyn/issues?q=is%3Aopen+is%3Aissue+label%3AArea-Compilers+label%3A%22Investigation+Required%22)
- [Investigating](https://github.com/dotnet/roslyn/issues?q=is%3Aopen+is%3Aissue+label%3AArea-Compilers+label%3AInvestigating)
- [Feature](https://github.com/dotnet/roslyn/issues?q=is%3Aopen+is%3Aissue+label%3AArea-Compilers+label%3A%22Feature+Request%22)
- [Question](https://github.com/dotnet/roslyn/issues?q=is%3Aopen+is%3Aissue+label%3AArea-Compilers+label%3AQuestion)
- [Test](https://github.com/dotnet/roslyn/issues?q=is%3Aopen+is%3Aissue+label%3AArea-Compilers+label%3ATest)
- [Concept-Design Debt](https://github.com/dotnet/roslyn/issues?q=is%3Aopen+is%3Aissue+label%3AArea-Compilers+label%3A%22Concept-Design+Debt%22)
- [Story](https://github.com/dotnet/roslyn/issues?q=is%3Aopen+is%3Aissue+label%3AArea-Compilers+label%3AStory)
- [Documentation](https://github.com/dotnet/roslyn/issues?q=is%3Aopen+is%3Aissue+label%3AArea-Compilers+label%3ADocumentation)
- [Need More Info](https://github.com/dotnet/roslyn/issues?q=is%3Aopen+is%3Aissue+label%3AArea-Compilers+label%3A%22Need+More+Info%22)
- [Blocked](https://github.com/dotnet/roslyn/issues?q=is%3Aopen+is%3Aissue+label%3AArea-Compilers+label%3ABlocked)
- [Discussion](https://github.com/dotnet/roslyn/issues?q=is%3Aopen+is%3Aissue+label%3AArea-Compilers+label%3ADiscussion)
