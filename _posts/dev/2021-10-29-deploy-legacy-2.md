---
layout: post
title: asp.net legacy system에 vuejs & nodejs system 적용하기(2탄)
categories: [dev]
author: 노혜진
email: hjroh@jinhakapply.com
date: 2022-04-01
tag:
  - asp.net
  - .net
  - IIS
  - legacy integration with javascript open source frameworks
  - nodejs
  - vue.js
---

지난 1탄에서 .NET 과 node를 통합하는 Architecture와 화면 통합을 위한 UI 측면에서 구조적으로 설명을 했다면  
2탄에서는 구현 sources 및 deploy 방법에 대해 정리해 보겠습니다.

## .NET WebForm Application 과 .NET MVC Appication 인증 공유

Legacy ASP.NET WebForm Applcation과 분리된 .NET MVC Application 프로젝트를 분리 구성한 형태에서 아래 사항을 고려해야 했습니다.

```
- 기존 legacy에서 인증된 사용자는 .NET MVC Application 에서도 추가 인증없이 사용 (기존의 asp.NET Forms 인증을 그대로 사용)
- 사용자 유효성 및 메뉴별 권한 체크 방식 동일하게 적용
```

기존 legacy에서는 domain form authentication 방식을 사용하고 있습니다.  
두개의 프로젝트가 윈폼 인증 정보를 공유하도록 하려면 몇가지 처리가 필요합니다.

1. 하나의 사이트에 MVC 소스를 가상폴더 대신 사이트 하위에 응용프로그램으로 설정했습니다. 라이브러리가 분리되어 있고 별도의 web.config를 구성해야 했기 떄문입니다.
2. 기존 legacy와 신규 mvc 두 응용프로그램이 동일한 worker process를 사용하도록 했습니다. 즉 IIS에서 사이트 셋팅시 동일한 어플리케이션 풀(application pool)을 사용하도록 셋팅합니다.
3. 두개 응용프로그램 web.config의 설정 값을 아래와 같이 적용했습니다.
   - authentication machine validationKey, decryptionKey값과 validation, decryption 알고리즘을 동일하게 셋팅해햐 합니다. form 인증시 사용하는 암호화 키과 알고리즘을 맞춰야 한다는 거는 추가 설명이 필요없을것 같네요
   - authentication 설정값에 compatibilityMode="Framework20SP1" 을 추가로 정의해 주어야 합니다.

```C#
<authentication mode="Forms">
  <forms name="TestAuth" defaultUrl="~/" path="/" loginUrl="/Login.aspx" protection="All" enableCrossAppRedirects="false" />
</authentication>
<machineKey validationKey="~~" decryptionKey="~~" validation="SHA2" decryption="3DES" compatibilityMode="Framework20SP1" />
```

## .NET MVC Project에서 사용자 인증 validation 및 오류 처리

기존 legacy에서 인증여부를 확인하고 권한 체크 등 기본적인 제어를 다수의 page에서 공통으로 사용하기 위해 기본 Page를 만들어 사용하고 있습니다.  
적용 환경(development, test, production)별 처리도 여기에 포함됩니다.

```C#
public class AuthenticatedPage : System.Web.UI.Page
```

MVC 프로젝트에서는 Page기반 동작이 불가하므로 Filters을 추가하였습니다.  
그리고 request별로 상용자 인증 및 권한 확인뿐만 아니라 인증 만료되지 않도록 인증 유효기간도 늘어나야 하기 때문에 새로운 폼 인증을 발행하도록 코드를 추가했습니다.  
중복된 작업이기 하지만 Bizness 공통 모듈로 처리해서 library를 공유해 쓰면 중복 소스는 많이 줄일 수 있습니다.

```C#
public class AuthFilter : ActionFilterAttribute, IActionFilter

  public override void OnActionExecuting(ActionExecutingContext filterContext)
    {
    ~

    FormsAuthenticationTicket ticket = new FormsAuthenticationTicket(
                                TicketVersion,													// Ticket version
                                UserID,												          // Login ID associated with ticket
                                DateTime.Now,										        // Date/time issued
                                NextExpire,		                          // Date/time to expire
                                false,							                    // "true" for a persistent user cookie
                                oldTicket.UserData,											// User-data, in this case the roles
                                FormsAuthentication.FormsCookiePath			// Path cookie valid for
                            );
                string hash = FormsAuthentication.Encrypt(ticket);
                HttpCookie cookie = FormsAuthentication.GetAuthCookie(FormsAuthentication.FormsCookieName, true);
                cookie.Value = hash;
                cookie.Domain = "test.com";
                HttpContext.Current.Response.Cookies.Set(cookie);
```

## .NET MVC Project에서 사용자 UI 화면 처리

Vue로 개발된 UI를 호출하는 경우와 데이터를 요청하는 Rest API request가 .NET MVC 응용프로그램으로 요청되면 설정된 Filter를 거친 후  
RouteConfig 를 통해 Controller 에러 라우팅 처리를 하였습니다.

```C#
public class RouteConfig
    {
        public static void RegisterRoutes(RouteCollection routes)
        {
          //Rest API 호출시  -> node api 서버로 호출
          routes.MapRoute(
              name: "NodeAPI",
              url: "api/{*subpath}",
              defaults: new { controller = "NodeAPI", action = "Index", subpath = UrlParameter.Optional }
            );

          // 특정 메뉴 호출시 -> vue bundle 된 javascript 소스 다운로드 하는 VIEW를 회신
          routes.MapRoute(
                name: "vue",
                url: "m/{*subpath}",
                defaults: new { controller = "Vue", action = "Index", id = UrlParameter.Optional }
            );


          ~
```

사용자가 호출하는 URL path(예. www.test.com/vue) 로 호출시 IIS에 mvc 응용프로그램의 경로를 해당 경로에 동작하게 하고
RouteConfig에서 api를 호출하는 경로를 제외하고 Vue로 개발한 소스를 호출하는 html 페이지를 다운받로록 VIEW를 매핑하도록 하였습니다.

```C#
 public class VueController : Controller
  {
   ~

    MenuBiz biz = new MenuBiz();
    DataSet Menus = biz.GetMenus(this._UserID);
    ViewData["ParentMenus"] = Menus.Tables[0];
    ViewData["ChildMenus"] = Menus.Tables[1];
    //선택된 메뉴 확인
    foreach (DataRow row in Menus.Tables[1].Rows)
    {
        if (String.Equals((string)Request.Url.AbsolutePath, (string)row["MenuURL"], StringComparison.OrdinalIgnoreCase))
        {
            SelectedUPMenuID = row["MenuUPID"].ToString();
            SelectedMenuName = row["MenuName"].ToString();
            break;
        }
    }
    //선택된 하위 메뉴의 상단 메뉴명을 확인
    foreach (DataRow row in Menus.Tables[0].Rows)
    {
        if (String.Equals(SelectedUPMenuID, row["MenuID"].ToString()))
        {
            SelectedUPMenuName = row["MenuName"].ToString();
            break;
        }
    }
    //선택된 상단메뉴 -> 화면에서 메뉴를 펼쳐지게 처리
    ViewBag.SelectedUPMenuID = SelectedUPMenuID;

    return View("Main");

```

.NET MVC VIEW는 Layout을 사용하였고 상단과 좌측메뉴를 구성하고 나머지 주요 본문의 내용은 VUE bundle 된 app.js 파일을 다운로드 받도록 구성하였습니다.
`<link href="~/Content/js/app.js" rel="preload" as="script">`
VIEW에 구성할 동적인 데이터는 Controller에서 ViewBag과 ViewData에 저장하여 사용하였습니다.

![deploy_screen.png](/assets/img/posts/dev/2021-10-29-deploy-legacy/deploy_screen.png)

- \_ViewStart.cshtml

```
@{
    Layout = "~/Views/Shared/_Layout.cshtml";
}
```

- Shared/\_Layout.cshtml

```C#
@using System.Data
@model DataSet

<!DOCTYPE html>
<html>
<head>
    <meta charset="utf-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>@ViewBag.Title</title>
    <link type="text/css" rel="stylesheet" href="/inc/Css/menu.css">
    <link href="~/Content/js/chunk-vendors.js" rel="preload" as="script">
    <link href="~/Content/js/app.js" rel="preload" as="script">

</head>

<body id="test_body">
    <div id="test_wrap">
        <div id="header">
            <h1>레거시 통합 프로젝트</h1>
            <span class="user">
                @{ if (HttpContext.Current.User.Identity.IsAuthenticated)
                 {
                     if (HttpContext.Current.User.Identity.Name != null)
                     {
                        <strong>@HttpContext.Current.User.Identity.Name 로그인 중:(<a href="/Logout.aspx">로그아웃</a>) </strong>
                     }
                 }
                 else
                 {
                    <a href="/Login.aspx">로그인</a>
                 }
                }

                (@ViewBag.ServerIP)
            </span>
        </div>
        <div id="contents">
            <!-- Left Area -->
            @{Html.RenderPartial("~/Views/Base/Menu.cshtml");}


            <div style="DISPLAY: none">
                <input type="text" id="txtMenuDB" value="1" />
                <input type="text" id="txtFirstPage" value="" />
            </div>

            @*선택된 메뉴의 VUE 화면영역*@
            @RenderBody()

        </div>
    </div>

</body>
```

- Menu.cshtml

```C#
using System.Data

<!-- Left Area -->
<div id="left">
    <ul id="leftmenu">
        @if (ViewData["ParentMenus"] != null && ViewData["ChildMenus"] != null)
        {
            int i = 0;
            var dtParent = ViewData["ParentMenus"] as DataTable;
            var dtChild = ViewData["ChildMenus"] as DataTable;

            foreach (DataRow row in dtParent.Rows)
            {
                DataRow[] fRows = dtChild.Select("MenuUPID = " + row["MenuID"].ToString(), "MenuSort");
                @*상단메뉴 클릭시 하단 메뉴 중 첫번쨰 메뉴를 호출하도록 처리 *@
                string url = "";
                if (row["MenuURL"] == null || row["MenuURL"] == "") { url = fRows[0]["MenuURL"].ToString(); }
                else { url = row["MenuURL"].ToString(); }
                <li id="@("m2_" + row["MenuID"])">
                    <a href="@url">@row["MenuName"]</a>
                    @*하단 메뉴가 하나인 경우 하단 메뉴 없이 상위 메뉴만 표시 *@
                    @if (fRows.Length > 1)
                    {
                        <ul>
                            @foreach (DataRow row2 in fRows )
                            {
                              <li><a href="@(row2["MenuURL"]) ">@row2["MenuName"]</a></li>
                            }
                        </ul>
                    }
                </li>
                i++;
            }
        }

     </ul>

</div>

<script type="text/javascript">
    @*호출한 url과 맞는 상단메뉴를 펼치게 처리 *@
    var selectedm2 = document.getElementById('@("m2_" + ViewBag.SelectedUPMenuID)');
    if (selectedm2) selectedm2.className = "selected"

</script>
```

## .NET MVC Project에서 Request api 처리

api 호출의 경우는 request method 별로 구분해서 작업을 하였고 api처리 앞단에 위치한 api gateway로 라우팅 되게 처리하였습니다.
api gateway 전송시 정의된 사용자 정의 header 값을 추가하였고 요청된 request url을 gateway url 포맷에 맞추어 redirect 하는 작업을 하엿습니다. \
나머지 header 정보(ContentType 등) 및 body는 그대로 전송하였습니다.

Controller는 최종 ActionResult로 회신하게 처리하였습니다.  
특히 GET방식의 요청의 경우 api서버로 부터 응답 받은 데이터는 일반적으로 json이어서 ContentResult로 회신을 하지만 파일 다운로드의 경우는 FileStreamResult로 회신되게 하여야합니다.

```C#
public class NodeAPIController : Controller
    {
        [AcceptVerbs(HttpVerbs.Get | HttpVerbs.Post | HttpVerbs.Put| HttpVerbs.Delete | HttpVerbs.Options | HttpVerbs.Head)]
        public ActionResult Index()
        {
          ~

          if (method == "GET")
            {
                HttpWebRequest webReq = (HttpWebRequest)HttpWebRequest.Create(APIBaseUrl + addUrl);
                HttpWebResponse webResp = (HttpWebResponse)webReq.GetResponse();

                //json 형태의 응답인 경우
                if (webResp.ContentType.ToString() == "application/json")
                {
                    StreamReader reader = new StreamReader(webResp.GetResponseStream());
                    reader.BaseStream.Position = 0;
                    string data = reader.ReadToEnd();

                    return new ContentResult { Content = data, ContentType = "application/json" };
                }
                //파일 형태의 응답인 경우
                else
                {
                    Stream stream = null;
                    stream = webResp.GetResponseStream();

                    return new FileStreamResult(stream, webResp.ContentType.ToString());
                }
```

파일 업로드시는 .NET Framewrok4.5 문제로 HttpWebRequest 대신 HttpClient 를 사용하였습니다.
파일명이 한글명일 경우 파일명이 깨지문 문제가 있어 unicode로 인코딩해서 해서 MultipartFormDataContent로 구성하여 api gateway에 요청전달 했습니다.

```C#
         ~
        if (method == "POST" && Request.ContentType.IndexOf("multipart") > -1 && Request.Files.Count > 0)
          {
              using (var content = new MultipartFormDataContent())
              {
                  var file = Request.Files[0];

                  //한글 파일명은 인코딩 처리
                  var unicodeFileName = file.FileName;
                  UTF8Encoding utf8 = new UTF8Encoding();
                  byte[] bytes = utf8.GetBytes(unicodeFileName);
                  char[] chars = new char[bytes.Length];
                  for (int index = 0; index < bytes.Length; index++)
                  {
                      chars[index] = Convert.ToChar(bytes[index]);
                  }
                  string fileName = new string(chars);

                  var fileContent = new StreamContent(file.InputStream);
                  fileContent.Headers.ContentType = new MediaTypeHeaderValue(file.ContentType);
                  content.Add(fileContent, "fileupload", fileName);
                  httpResp = client.PostAsync(addUrl, content).Result;
```

저희가 구성한 node api 서버는 응답 회신 status code를 500 에러로 회신하는 경우에도 body에 json을 전송하는 경우가 있습니다.
ASP.NET에서는 기본적으로 응답코드 200번대를 제외하고는 body에 내용을 전달할수 없습니다.
api서버에서 받은 500에러를 그대로 회신하면 에러가 발생합니다.
회신 500 에러 body에 내용을 전달하려면 mvc 응용프로그램 web.config설정 파일에서 system.webServer설정에 PassThrough를 추가로 셋팅해야 합니다.

```C#
  <system.webServer>
    <httpErrors errorMode="DetailedLocalOnly" existingResponse="PassThrough" />
  </system.webServer>
```

mvc 응용프로그램 소스는 추후 비지니스가 바뀌더라도 처음 적용후 수정이 필요치 않습니다.
하지만 계속 추가되고 변경되는 비지니스로 인하여 Vue 소스 배포는 자주 발생되게됩니다. VUE 소스는 build 후 transform된 js소스를 windows IIS서버에 배포해야 합니다.
이를 위해 devops 배포 프로세스로 활용중인 jenkins를 사용하도록 하였습니다.
jenkins 에서는 git서버에 배포용 버전으로 tagging된 소스를 기반으로 최신 모듈을 다운받고 build 한후 빌드된 최종 소스는 IIS에 설정된 경로로 js 파일을 복사하도록 자동처리하였습니다.

legacy 시스템이 오래전 환경으로 구축되어 있어서 호환성을 유지하는데 제한적인 요소가 많고 문제 해결을 위한 자료를 찾는데 어려움이 있었습니다.
하지만 안정적으로 운영중인 프로젝트 결과물들도 새로운 기술들을 적용하는데 허들이 되지 않도록 통합을 하고자 하는 분들이 많을 것으로 생각됩니다.
문제풀이는 잼있습니다. 같이 힘내보죠!
