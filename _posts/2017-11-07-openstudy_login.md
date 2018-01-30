---
layout: post
title: 'Open-Study Login'
author: HanSooKyeong
date: 2017-11-07 13:21
tags: [project]
image: /files/covers/blog.jpg
---

#. login 구현

* #### 로그인 테이블 만들기
  - 회원테이블 추가
``` java
CREATE TABLE user (
  userNo int(11) NOT NULL AUTO_INCREMENT,
  userEmail varchar(50) NOT NULL,
  userPwd varchar(50) DEFAULT NULL
)
```

* #### 로그인창 jsp 만들기
  - 로그인폼 구현 (추후 구현 - 아이디저장, 자동로그인)

``` html
<form action="/login/loginPost" method="post" id="loginForm">
	<div class="form-group label-floating">
		<label class="control-label">이메일 주소</label> <input name="userEmail" type="text" id="exampleInputEmail1" class="form-control" value="${cookie.rememberID.value}">
	</div>
	<div class="form-group label-floating">
		<label class="control-label">비밀번호</label> <input name="userPwd" type="password" id="exampleInputPassword1" class="form-control">
	</div>
	<div class="checkbox">
		<label> <input type="checkbox" name="rememberEmail"> 아이디저장 </label>
		<label> <input type="checkbox" name="useCookie"> 자동로그인 </label>
	</div>
	<button type="submit" class="btn btn-default">로그인</button>
</form>
```

* #### 필요한 pom.xml에 메이븐파일 작성
  - Mybatis 를 사용하기 위해 아래 4개 메이븐파일을 pom.xml에 추가

```
 <!-- jdbc 연결 -->
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-jdbc</artifactId>
			<version>${org.springframework-version}</version>
		</dependency>

		<!-- mysql -->
		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
			<version>5.1.41</version>
		</dependency>

		<!-- mybatis -->
		<dependency>
			<groupId>org.mybatis</groupId>
			<artifactId>mybatis</artifactId>
			<version>3.4.1</version>
		</dependency>

		<!-- mybatis-spring 연결 -->
		<dependency>
			<groupId>org.mybatis</groupId>
			<artifactId>mybatis-spring</artifactId>
			<version>1.3.0</version>
		</dependency>
```

* #### 로그인 Controller 작성
  - loginGet 메서드 request.getHeader("referer") - 이전 페이지 기억
  - dest 세션은 기존 페이지URI 를 저장

 ``` java

 @Controller
 @RequestMapping("/login")
 public class LoginController {

 	@Inject
 	private BCryptPasswordEncoder pwdEncoder;

 	@Inject
 	private UserService userService;

 	@RequestMapping(value = "/loginGet", method = RequestMethod.GET)
 	public String loginGet(HttpServletRequest request) throws Exception { // 로그인창
 		HttpSession session = request.getSession();
 		if (request.getRequestURI() != null && request.getHeader("referer") != null) { // 바로 로그인 URI 접근시 null값 방지
 			if (session.getAttribute("temp") != null) // auth인터셉터 거칠때는 이전 페이지 저장 X
 				session.removeAttribute("temp");
 			else if (!(request.getRequestURI().equals("/login/loginGet") && request.getHeader("referer").equals("http://localhost/login/loginGet"))) // 로그인창 여러번 클릭 할때는 저장하지 않음
 				session.setAttribute("dest", request.getHeader("referer")); // 이전 페이지 정보 저장
 		}

 		return "/user/login";
 	}

 	@RequestMapping(value = "/loginPost", method = RequestMethod.POST)
 	public void loginPost(LoginDTO dto, Model model, HttpSession session) throws Exception { // 로그인

     UserVO vo = null;

 		if (pwdEncoder.matches(dto.getUserPwd(), userService.getPwd(dto))) // DB 비밀번호와 로그인 비밀번호 비교
 			vo = userService.login(dto); // vo에 userNo, userEmail, UserNick, userAuth 저장
 		else { // 로그인 실패시
 			model.addAttribute("loginFail", true);
       return;
 		}

 		model.addAttribute("userVO", vo); // model에 {userVO : vo} 저장

 	}

 	@RequestMapping(value = "/logout", method = RequestMethod.GET)
 	public String logout(HttpServletRequest request, HttpServletResponse response, HttpSession session) throws Exception { // 로그아웃
 		Object obj = session.getAttribute("login");

 		if (obj != null) {
 			UserVO vo = (UserVO) obj;
 			session.removeAttribute("login"); // 세션 제거
 			session.invalidate();
 		}
 		return "/login/logout";
 	}

 ```

* #### loginInterCepter.class 작성
  - servlet-context.xml 에 아래내용을 추가

```
 <beans:bean id="loginInterceptor" class="패키지명.LoginInterceptor" />

  <interceptors>
		<interceptor>
			<mapping path="/login/loginPost" />
			<beans:ref bean="loginInterceptor" />
		</interceptor>
	 </interceptors>
```

  - 그리고나서 HandlerInterceptorAdapter 를 상속받으면 인터셉터를 사용
  - preHandle은 컨트롤러를 거치기 전이고, postHandle은 컨트롤러를 거친 후


```

public class LoginInterceptor extends HandlerInterceptorAdapter {
	private static final String LOGIN = "login";

	@Override
	public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
		HttpSession session = request.getSession();
		if (session.getAttribute(LOGIN) != null) // 기존 로그인정보 삭제
			session.removeAttribute(LOGIN);

		return true;
	}

	@Override
	public void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler, ModelAndView modelAndView) throws Exception {
		HttpSession session = request.getSession();
		ModelMap modelMap = modelAndView.getModelMap();
		Object userVO = modelMap.get("userVO"); // userVO 모델속성 가져옴
		UserVO vo = (UserVO) userVO;

		response.setContentType("text/html; charset=UTF-8");
		PrintWriter out = response.getWriter();

		if (modelMap.get("loginFail") != null) { // 아이디,비밀번호가 일치하지 않을 경우
			out.println("<script>alert('아이디와 비밀번호가 일치하지 않습니다.'); location.href = '/login/loginGet'</script>"); // 실패알림 후 로그인창 가기
			out.close();
			return;
		}

		if (userVO != null) { // 로그인 성공시(객체가 있을 경우)
			session.setAttribute(LOGIN, userVO); // userVO 세션저장

			String dest = (String) session.getAttribute("dest"); // URI 세션 저장

			if (dest.equals("http://localhost/login/loginPost")) // 아이디 비밀번호가 실패할 경우 다시 로그인할때 loginPost 저장하기떄문에 방지
				dest = "/";
			response.sendRedirect(dest != null ? (String) dest : "/");
		} else
			response.sendRedirect("/login/loginGet");

	}

}

```

* #### userMapper.xml 작성
  - 로그인 userNo, UserEamil, userNick, userAuth (유저권한) 가져옴

 ```
 <select id="login" resultType="UserVO"> <!-- 로그인 -->
		<![CDATA[select userNo, userEmail, userNick, userAuth from user where userEmail = #{userEmail}]]>
	</select>

 ```

 * #### userDAO, userService 작성
  - DAO는 DB등 하나의 데이터 접근 및 갱신만 처리하며, Service 는 DAO들을 호출하여 읽은 데이터에 대한 비지니스 로직을 수행하며 트랜잭션으로 처리함

 ``` java

  - userDAO
 @Repository
 public class UserDAOImpl implements UserDAO {

 	@Inject
 	private SqlSession session;

 	private static String namespace = "org.sbang.mapper.UserMapper";

 	@Override
 	public UserVO login(LoginDTO dto) throws Exception { // 로그인
 		return session.selectOne(namespace + ".login", dto);
 	}

  -userService
  @Service
public class UserServiceImpl implements UserService {

	@Inject
	private UserDAO userDAO;

	@Override
	public UserVO login(LoginDTO dto) throws Exception { // 로그인
		return userDAO.login(dto);
	}

 ```
