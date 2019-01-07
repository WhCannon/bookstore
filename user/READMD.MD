# 用户模块
  - [User](#user)
  - [UserDao](#userdao)
  - [UserService](#userservice)
  - [UserServlet](#userservlet)
  - [扩展：消息格式化](#消息格式化)
  
  
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
* 消息格式化
```java
public class Demo1 {
	//消息格式化
	public void fun1() {
		/* MessageFormat.format(模板,参数)：需要给出模板
		 * 包含了占位符的字符串就是模板！
		 * 占位符：{0}、{1}、{2}...
		 * 参数是可变的，需要指定模板中占位符的值！有几个占位符就要提供几个参数
		 */
		String s = MessageFormat.format("{0}或{1}错误！", "用户名", "密码");
		System.out.println(s);
	}
}
```
