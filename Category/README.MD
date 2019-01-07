# 分类模块
  -[domain](#Category)
  -[dao](#CategoryDao)
  -[service](#CategoryService)
  -[servlet](#CategoryServlet)
  
  
## Category
```java
public class Category {
	private String cid;
	private String cname;
	public String getCid() {
		return cid;
	}
	public void setCid(String cid) {
		this.cid = cid;
	}
	public String getCname() {
		return cname;
	}
	public void setCname(String cname) {
		this.cname = cname;
	}
	public String toString() {
		return "Category [cid=" + cid + ", cname=" + cname + "]";
	}
}
```

## CategoryServlet
```java
public class CategoryServlet extends BaseServlet {
	private CategoryService categoryService = new CategoryService();
	// 查询所有分类
	public String findAll(HttpServletRequest req, HttpServletResponse resp)
			throws ServletException, IOException {
		req.setAttribute("categoryList", categoryService.findAll());
		return "f:/jsps/left.jsp";
	}
}
```

## CategoryService
```java
public class CategoryService {
	private CategoryDao categoryDao = new CategoryDao();
	 // 查询所有分类
	public List<Category> findAll() {
		return categoryDao.findAll();
	} 
}
```

## CategoryDao
```java
public class CategoryDao {
	private QueryRunner qr = new TxQueryRunner();
	 // 查询所有分类
	public List<Category> findAll() {
		try {
			String sql = "select * from category";
			return qr.query(sql, new BeanListHandler<Category>(Category.class));
		} catch (SQLException e) {
			throw new RuntimeException(e);
		}
	}
}
```
