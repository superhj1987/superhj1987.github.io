---
layout: post
title: "关于Spring mvc中modelattribute无法制定别名的解决方案"
date: 2015-01-21 12:00:00 +0800
comments: true
categories: 
---

最近由于项目需要，发现spring mvc在绑定参数时有这么一个缺陷。
  
**Url**: http://localhost:8080/api/test?user_name=testUser  

**Controller**:   
<pre>
@Controller
@RequestMapping("/api")
public class ApiController extends BaseController {

    @RequestMapping(value = "/test", headers = "Accept=application/json")
    public void authUser(ModelMap modelMap, Account acc) {
        ResultPack.packOk(modelMap);
    }
}

public class Account{
    private static final long serialVersionUID = 750752375611621980L;

    private long id;
    private String userName;
    private String password;
    private AccountType type = AccountType.ADMIN;
    private long timeTag;
    private int status = 1;
    ...
    ...
}
</pre>

user_name无法映射到acc的userName上。如果使用json的方式，可以使用JsonProperty注解来解决。否则，spring貌似没提供解决方案。

于是追踪了一下spring mvc的源代码，发现可以通过重写ServletModelAttributeMethodProcessor来支持这个功能。  

**Github**:  https://github.com/superhj1987/spring-mvc-model-attribute-alias