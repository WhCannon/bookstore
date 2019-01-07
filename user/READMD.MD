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


```

## UserServlet
```java

```
