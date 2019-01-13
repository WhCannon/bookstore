# 分类模块
  - [domain](#category)
  - [dao](#categorydao)
  - [service](#categoryservice)
  - [servlet](#categoryservlet)
  - [后台分类管理](#admincategoryservlet)
  
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
	private BookDao bookDao = new BookDao();
	 // 查询所有分类
	public List<Category> findAll() {
		return categoryDao.findAll();
	}

	 // 添加分类
	public void add(Category category) {
		categoryDao.add(category);
	}

	 // 删除分类
	public void delete(String cid) throws CategoryException {
		// 获取该分类下图书的本数
		int count = bookDao.getCountByCid(cid);
		// 如果该分类下存在图书，不让删除，我们抛出异常
		if(count > 0) throw new CategoryException("该分类下还有图书，不能删除！");
		// 删除该分类
		categoryDao.delete(cid);
	}

	 // 加载分类
	public Category load(String cid) {
		return categoryDao.load(cid);
	}

	 // 修改分类
	public void edit(Category category) {
		categoryDao.edit(category);
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

	 // 添加分类
	public void add(Category category) {
		try {
			String sql = "insert into category values(?,?)";
			qr.update(sql, category.getCid(), category.getCname());
		} catch (SQLException e) {
			throw new RuntimeException(e);
		}
	}

	 // 删除分类
	public void delete(String cid) {
		try {
			String sql = "delete from category where cid=?";
			qr.update(sql, cid);
		} catch (SQLException e) {
			throw new RuntimeException(e);
		}
	}

	 // 加载分类
	public Category load(String cid) {
		try {
			String sql = "select * from category where cid=?";
			return qr.query(sql, new BeanHandler<Category>(Category.class), cid);
		} catch (SQLException e) {
			throw new RuntimeException(e);
		}
	}

	 // 修改分类
	public void edit(Category category) {
		try {
			String sql = "update category set cname=? where cid=?";
			qr.update(sql, category.getCname(), category.getCid());
		} catch (SQLException e) {
			throw new RuntimeException(e);
		}
	}
}
```

## AdminCategoryServlet
```java
public class AdminCategoryServlet extends BaseServlet {
	private CategoryService categoryService = new CategoryService();
	
	 // 修改分类
	public String edit(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		/*
		 * 1. 封装表单数据
		 * 2. 调用service方法完成修改工作
		 * 3. 调用findAll
		 */
		Category category = CommonUtils.toBean(request.getParameterMap(), Category.class);
		categoryService.edit(category);
		return findAll(request, response);
	}
	
	 // 修改之前的加载工作
	public String editPre(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		String cid = request.getParameter("cid");
		request.setAttribute("category", categoryService.load(cid));
		return "f:/adminjsps/admin/category/mod.jsp";
	}
	
	 // 删除分类
	public String delete(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		/*
		 * 1. 获取参数:cid
		 * 2. 调用service方法，传递cid参数
		 *   > 如果抛出异常，保存异常信息，转发到msg.jsp显示
		 * 3. 调用findAll()
		 */
		String cid = request.getParameter("cid");
		try {
			categoryService.delete(cid);
			return findAll(request, response);
		} catch(CategoryException e) {
			request.setAttribute("msg", e.getMessage());
			return "f:/adminjsps/msg.jsp";
		}
	}
	
	 // 添加分类
	public String add(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		/* 1. 封装表单数据
		 * 2. 补全：cid
		 * 3. 调用service方法完成添加工作
		 * 4. 调用findAll() */
		Category category = CommonUtils.toBean(request.getParameterMap(), Category.class);
		category.setCid(CommonUtils.uuid());
		categoryService.add(category);
		return findAll(request, response);
	}
	
	//查询所有分类
	public String findAll(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		/*
		 * 1. 调用service方法，得到所有分类
		 * 2. 保存到request域，转发到/adminjsps/admin/category/list.jsp
		 */
		request.setAttribute("categoryList", categoryService.findAll());
		return "f:/adminjsps/admin/category/list.jsp";
	}
}
```
