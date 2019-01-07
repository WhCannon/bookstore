# 用户模块
  - [User](#user)
  - [UserDao](#userdao)
  - [UserService](#userservice)
  - [UserServlet](#userservlet)
  
  
## User
```java
// User的领域对象
public class User {
	// 对应数据库表
	private String uid;// 主键
	private String username;// 用户名
	private String password;// 密码
	private String email;// 邮箱
	private String code;// 激活码
	private boolean state;// 状态（已激活和未激活）
  getters和setters
}
```

## UserDao
```java
// User持久层
public class UserDao {
	private QueryRunner qr = new TxQueryRunner();
	// 按用户名查询
	public User findByUsername(String username) {
		try {
			String sql = "select * from tb_user where username=?";
			return qr.query(sql, new BeanHandler<User>(User.class), username);
		} catch(SQLException e) {
			throw new RuntimeException(e);
		}
	}
	
	// 按邮箱查询
	public User findByEmail(String email) {
		try {
			String sql = "select * from tb_user where email=?";
			return qr.query(sql, new BeanHandler<User>(User.class), email);
		} catch(SQLException e) {
			throw new RuntimeException(e);
		}
	}
	
	// 插入User
	public void add(User user) {
		try {
			String sql = "insert into tb_user values(?,?,?,?,?,?)";
			Object[] params = {user.getUid(), user.getUsername(), 
					user.getPassword(), user.getEmail(), user.getCode(),
					user.isState()};
			qr.update(sql, params);
		} catch(SQLException e) {
			throw new RuntimeException(e);
		}
	}
	
	// 按激活码查询
	public User findByCode(String code) {
		try {
			String sql = "select * from tb_user where code=?";
			return qr.query(sql, new BeanHandler<User>(User.class), code);
		} catch(SQLException e) {
			throw new RuntimeException(e);
		}
	}
	
	// 修改指定用户的指定状态
	public void updateState(String uid, boolean state) {
		try {
			String sql = "update tb_user set state=? where uid=?";
			qr.update(sql, state, uid);
		} catch(SQLException e) {
			throw new RuntimeException(e);
		}}}
```

## UserService
```java
public class UserException extends Exception {//自定义异常
	public UserException() {
		super();
	}
	public UserException(String message) {
		super(message);
	}
}
// User业务层
public class UserService {
	private UserDao userDao = new UserDao();
	
	 // 注册功能
	public void regist(User form) throws UserException{
		// 校验用户名
		User user = userDao.findByUsername(form.getUsername());
		if(user != null) throw new UserException("用户名已被注册！");
		
		// 校验email
		user = userDao.findByEmail(form.getEmail());
		if(user != null) throw new UserException("Email已被注册！");
		
		// 插入用户到数据库
		userDao.add(form);
	}
	
	// 激活功能
	public void active(String code) throws UserException {
		// 1. 使用code查询数据库，得到user
		User user = userDao.findByCode(code);
		// 2. 如果user不存在，说明激活码错误
		if(user == null) throw new UserException("激活码无效！");
		// 3. 校验用户的状态是否为未激活状态，如果已激活，说明是二次激活，抛出异常
		if(user.isState()) throw new UserException("您已经激活过了，不要再激活了，除非你想死！");
		// 4. 修改用户的状态
		userDao.updateState(user.getUid(), true);
	}
	
	 // 登录功能
	public User login(User form) throws UserException {
		/*
		 * 1. 使用username查询，得到User
		 * 2. 如果user为null，抛出异常（用户名不存在）
		 * 3. 比较form和user的密码，若不同，抛出异常（密码错误）
		 * 4. 查看用户的状态，若为false，抛出异常（尚未激活）
		 * 5. 返回user
		 */
		User user = userDao.findByUsername(form.getUsername());
		if(user == null) throw new UserException("用户名不存在！");
		if(!user.getPassword().equals(form.getPassword()))
			throw new UserException("密码错误！");
		if(!user.isState()) throw new UserException("尚未激活！");
		return user;
	}
}
```

## UserServlet
```java
// User表述层
public class UserServlet extends BaseServlet {
	private UserService userService = new UserService();
	
	 // 退出功能
	public String quit(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		request.getSession().invalidate();
		return "r:/index.jsp";
	}
	
	public String login(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		/*
		 * 1. 封装表单数据到form中
		 * 2. 输入校验（不写了）
		 * 3. 调用service完成激活
		 *   > 保存错误信息、form到request，转发到login.jsp
		 * 4. 保存用户信息到session中，然后重定向到index.jsp
		 */
		User form = CommonUtils.toBean(request.getParameterMap(), User.class);
		try {
			User user = userService.login(form);
			request.getSession().setAttribute("session_user", user);
			return "r:/index.jsp";
		} catch (UserException e) {
			request.setAttribute("msg", e.getMessage());
			request.setAttribute("form", form);
			return "f:/jsps/user/login.jsp";
		}
	}
	
	 // 激活功能
	public String active(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		/*
		 * 1. 获取参数激活码
		 * 2. 调用service方法完成激活
		 *   > 保存异常信息到request域，转发到msg.jsp
		 * 3. 保存成功信息到request域，转发到msg.jsp
		 */
		String code = request.getParameter("code");
		try {
			userService.active(code);
			request.setAttribute("msg", "恭喜，您激活成功了！请马上登录！");
		} catch (UserException e) {
			request.setAttribute("msg", e.getMessage());
		}
		return "f:/jsps/msg.jsp";
	}
	
	 // 注册功能
	public String regist(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		/* 1. 封装表单数据到form对象中
		 * 2. 补全：uid、code
		 * 3. 输入校验
		 *   > 如果出错,保存错误信息和form到request域，转发到regist.jsp
		 * 4. 调用service方法完成注册
		 *   > 如果出错,保存错误信息、form到request域，转发到regist.jsp
		 * 5. 发邮件
		 * 6. 保存成功信息转发到msg.jsp */
		
		// 封装表单数据
		User form = CommonUtils.toBean(request.getParameterMap(), User.class);
		// 补全
		form.setUid(CommonUtils.uuid());
		form.setCode(CommonUtils.uuid() + CommonUtils.uuid());
		 // 输入校验
		 // 1. 创建一个Map，用来封装错误信息，其中key为表单字段名称，值为错误信息
		Map<String,String> errors = new HashMap<String,String>();
		 // 2. 获取form中的username、password、email进行校验
		String username = form.getUsername();
		if(username == null || username.trim().isEmpty()) {
			errors.put("username", "用户名不能为空！");
		} else if(username.length() < 3 || username.length() > 10) {
			errors.put("username", "用户名长度必须在3~10之间！");
		}
		
		String password = form.getPassword();
		if(password == null || password.trim().isEmpty()) {
			errors.put("password", "密码不能为空！");
		} else if(password.length() < 3 || password.length() > 10) {
			errors.put("password", "密码长度必须在3~10之间！");
		}
		
		String email = form.getEmail();
		if(email == null || email.trim().isEmpty()) {
			errors.put("email", "Email不能为空！");
		} else if(!email.matches("\\w+@\\w+\\.\\w+")) {
			errors.put("email", "Email格式错误！");
		}
		 // 3. 判断是否存在错误信息
		if(errors.size() > 0) {
			// 1. 保存错误信息
			// 2. 保存表单数据
			// 3. 转发到regist.jsp
			request.setAttribute("errors", errors);
			request.setAttribute("form", form);
			return "f:/jsps/user/regist.jsp";
		}
		
		 // 调用service的regist()方法
		try {
			userService.regist(form);
		} catch (UserException e) {
			/* 1. 保存异常信息
			 * 2. 保存form
			 * 3. 转发到regist.jsp */
			request.setAttribute("msg", e.getMessage());
			request.setAttribute("form", form);
			return "f:/jsps/user/regist.jsp";
		}
		
		// 发邮件
		// 获取配置文件内容
		Properties props = new Properties();
		props.load(this.getClass().getClassLoader()
				.getResourceAsStream("email_template.properties"));
		String host = props.getProperty("host");//获取服务器主机
		String uname = props.getProperty("uname");//获取用户名
		String pwd = props.getProperty("pwd");//获取密码
		String from = props.getProperty("from");//获取发件人
		String to = form.getEmail();//获取收件人
		String subject = props.getProperty("subject");//获取主题
		String content = props.getProperty("content");//获取邮件内容
		content = MessageFormat.format(content, form.getCode());//替换{0}
		//消息格式化
		/*public void fun1() {
			* MessageFormat.format(模板,参数)：需要给出模板
			 * 包含了占位符的字符串就是模板！
			 * 占位符：{0}、{1}、{2}...
			 * 参数是可变的，需要指定模板中占位符的值！有几个占位符就要提供几个参数
			 *
			String s = MessageFormat.format("{0}或{1}错误！", "用户名", "密码");
			System.out.println(s);
		} */
		Session session = MailUtils.createSession(host, uname, pwd);//得到session
		Mail mail = new Mail(from, to, subject, content);//创建邮件对象
		try {
			MailUtils.send(session, mail);//发邮件！
		} catch (MessagingException e) {
		}
		
		/* 1. 保存成功信息
		 * 2. 转发到msg.jsp */
		request.setAttribute("msg", "恭喜，注册成功！请马上到邮箱激活");
		return "f:/jsps/msg.jsp";
	}
}
```

