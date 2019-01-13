# 书模块
  - [domain](#book)
  - [servlet](#bookservlet)
  - [service](#bookservice)
  - [dao](#bookdao)
  - [后台图书管理](#adminbookservlet)
  - [后台添加上传图书](#adminaddbookservlet)

## Book
```java
public class Book {
	private String bid;
	private String bname;
	private double price;
	private String author;
	private String image;
	private Category category;
	public String getBid() {
		return bid;
	}
	public void setBid(String bid) {
		this.bid = bid;
	}
	public String getBname() {
		return bname;
	}
	public void setBname(String bname) {
		this.bname = bname;
	}
	public double getPrice() {
		return price;
	}
	public void setPrice(double price) {
		this.price = price;
	}
	public String getAuthor() {
		return author;
	}
	public void setAuthor(String author) {
		this.author = author;
	}
	public String getImage() {
		return image;
	}
	public void setImage(String image) {
		this.image = image;
	}
	public Category getCategory() {
		return category;
	}
	public void setCategory(Category category) {
		this.category = category;
	}
}
```

## BookDao
```java
public class BookDao {
	private QueryRunner qr = new TxQueryRunner();
	
	 // 查询所有图书
	public List<Book> findAll() {
		try {
			String sql = "select * from book where del=false";
			return qr.query(sql, new BeanListHandler<Book>(Book.class));
		} catch(SQLException e) {
			throw new RuntimeException(e);
		}
	}

	 // 按分类查询
	public List<Book> findByCategory(String cid) {
		try {
			String sql = "select * from book where cid=? and del=false";
			return qr.query(sql, new BeanListHandler<Book>(Book.class), cid);
		} catch(SQLException e) {
			throw new RuntimeException(e);
		}
	}

	 // 加载方法
	public Book findByBid(String bid) {
		try {
			 // 我们需要在Book对象中保存Category的信息
			String sql = "select * from book where bid=?";
			Map<String,Object> map = qr.query(sql, new MapHandler(), bid);
			 // 使用一个Map，映射出两个对象，再给这两个对象建立关系！
			Category category = CommonUtils.toBean(map, Category.class);
			Book book = CommonUtils.toBean(map, Book.class);
			book.setCategory(category);
			return book;
		} catch(SQLException e) {
			throw new RuntimeException(e);
		}
	}

	 // 查询指定分类下的图书本数
	public int getCountByCid(String cid) {
		try {
			String sql = "select count(*) from book where cid=?";
			Number cnt = (Number)qr.query(sql, new ScalarHandler(), cid);
			return cnt.intValue();
		} catch(SQLException e) {
			throw new RuntimeException(e);
		}
	}

	 // 添加图书
	public void add(Book book) {
		try {
			String sql = "insert into book values(?,?,?,?,?,?)";
			Object[] params = {book.getBid(), book.getBname(), book.getPrice(),
					book.getAuthor(), book.getImage(), book.getCategory().getCid()};
			qr.update(sql, params);
		} catch(SQLException e) {
			throw new RuntimeException(e);
		}
	}
	
	 // 删除图书
	public void delete(String bid) {
		try {
			String sql = "update book set del=true where bid=?";
			qr.update(sql, bid);
		} catch(SQLException e) {
			throw new RuntimeException(e);
		}
	}

	//修改图书
	public void edit(Book book) {
		try {
			String sql = "update book set bname=?, price=?,author=?, image=?, cid=? where bid=?";
			Object[] params = {book.getBname(), book.getPrice(),
					book.getAuthor(), book.getImage(), 
					book.getCategory().getCid(), book.getBid()};
			qr.update(sql, params);
		} catch(SQLException e) {
			throw new RuntimeException(e);
		}
	}
}
```

## BookService
```java
public class BookService {
	private BookDao bookDao = new BookDao();
	
	 // 查询所有图书
	public List<Book> findAll() {
		return bookDao.findAll();
	}

	 // 按分类查询图书
	public List<Book> findByCategory(String cid) {
		return bookDao.findByCategory(cid);
	}

	public Book load(String bid) {
		return bookDao.findByBid(bid);
	}

	 //　添加图书
	public void add(Book book) {
		bookDao.add(book);
	}
	
	public void delete(String bid) {
		bookDao.delete(bid);
	}

	public void edit(Book book) {
		bookDao.edit(book);
	}
}
```

## BookServlet
```java
public class BookServlet extends BaseServlet {
	private BookService bookService = new BookService();
	
	//查看图书信息
	public String load(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		/* 1. 得到参数bid
		 * 2. 查询得到Book
		 * 3. 保存，转发到desc.jsp */
		request.setAttribute("book", bookService.load(request.getParameter("bid")));
		return "f:/jsps/book/desc.jsp";
	}
	
	// 查询所有图书
	public String findAll(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		request.setAttribute("bookList", bookService.findAll());
		return "f:/jsps/book/list.jsp";
	}
	
	 // 按分类查询
	public String findByCategory(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		String cid = request.getParameter("cid");
		request.setAttribute("bookList", bookService.findByCategory(cid));
		return "f:/jsps/book/list.jsp";
	}
}
```

## AdminBookServlet
```java
public class AdminBookServlet extends BaseServlet {
	private BookService bookService = new BookService();
	private CategoryService categoryService = new CategoryService();
	
	 // 添加图书
	public String addPre(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		 // 查询所有分类，保存到request，转发到add.jsp
		 // add.jsp中把所有的分类使用下拉列表显示在表单中
		request.setAttribute("categoryList", categoryService.findAll());
		return "f:/adminjsps/admin/book/add.jsp";
	}
	
	public String load(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		/*
		 * 1. 获取参数bid，通过bid调用service方法得到Book对象
		 * 2. 获取所有分类，保存到request域中
		 * 3. 保存book到request域中，转发到desc.jsp
		 */
		request.setAttribute("book", bookService.load(request.getParameter("bid")));
		request.setAttribute("categoryList", categoryService.findAll());
		return "f:/adminjsps/admin/book/desc.jsp";
	}
	
	 // 查看所有图书
	public String findAll(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		request.setAttribute("bookList", bookService.findAll());
		return "f:/adminjsps/admin/book/list.jsp";
	}
	
	 // 删除图书
	public String delete(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		String bid = request.getParameter("bid");
		bookService.delete(bid);
		return findAll(request, response);
	}
	
	 // 修改图书
	public String edit(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		Book book = CommonUtils.toBean(request.getParameterMap(), Book.class);
		Category category = CommonUtils.toBean(request.getParameterMap(), Category.class);
		book.setCategory(category);
		
		bookService.edit(book);
		return findAll(request, response);
	}
}
```

## AdminAddBookServlet
上传不能使用BaseServlet
```java
public class AdminAddBookServlet extends HttpServlet {
	private BookService bookService = new BookService();
	private CategoryService categoryService = new CategoryService();
	
	public void doPost(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		request.setCharacterEncoding("utf-8");
		response.setContentType("text/html;charset=utf-8");
		
		 // 1. 把表单数据封装到Book对象中
		// 上传三步，创建工厂
		DiskFileItemFactory factory = new DiskFileItemFactory(15 * 1024, new File("F:/f/temp"));
		// 得到解析器
		ServletFileUpload sfu = new ServletFileUpload(factory);
		// 设置单个文件大小为15KB
		sfu.setFileSizeMax(20 * 1024);
		// 使用解析器去解析request对象，得到List<FileItem>
		try {
			List<FileItem> fileItemList = sfu.parseRequest(request);
			/*
			 * * 把fileItemList中的数据封装到Book对象中
			 *   > 把所有的普通表单字段数据先封装到Map中
			 *   > 再把map中的数据封装到Book对象中
			 */
			Map<String,String> map = new HashMap<String,String>();
			for(FileItem fileItem : fileItemList) {
				if(fileItem.isFormField()) {
					map.put(fileItem.getFieldName(), fileItem.getString("UTF-8"));
				}
			}

			Book book = CommonUtils.toBean(map, Book.class);
			// 为book指定bid
			book.setBid(CommonUtils.uuid());
			 // 需要把Map中的cid封装到Category对象中，再把Category赋给Book
			Category category = CommonUtils.toBean(map, Category.class);
			book.setCategory(category);
			
			/*
			 * 2. 保存上传的文件
			 *   * 保存的目录
			 *   * 保存的文件名称
			 */
			// 得到保存的目录
			String savepath = this.getServletContext().getRealPath("/book_img");
			// 得到文件名称：给原来文件名称添加uuid前缀！避免文件名冲突
			String filename = CommonUtils.uuid() + "_" + fileItemList.get(1).getName();
			 // 校验文件的扩展名
			if(!filename.toLowerCase().endsWith("jpg")) {
				request.setAttribute("msg", "您上传的图片不是JPG扩展名！");
				request.setAttribute("categoryList", categoryService.findAll());
				request.getRequestDispatcher("/adminjsps/admin/book/add.jsp")
						.forward(request, response);
				return;
			}
			
			// 使用目录和文件名称创建目标文件
			File destFile = new File(savepath, filename);
			// 保存上传文件到目标文件位置
			fileItemList.get(1).write(destFile);
			
			 // 3. 设置Book对象的image，即把图片的路径设置给Book的image
			book.setImage("book_img/" + filename);
			
			 // 4. 使用BookService完成保存
			bookService.add(book);
			
			 // 校验图片的尺寸
			Image image = new ImageIcon(destFile.getAbsolutePath()).getImage();
			if(image.getWidth(null) > 200 || image.getHeight(null) > 200) {
				destFile.delete();//删除这个文件！
				request.setAttribute("msg", "您上传的图片尺寸超出了200 * 200！");
				request.setAttribute("categoryList", categoryService.findAll());
				request.getRequestDispatcher("/adminjsps/admin/book/add.jsp")
						.forward(request, response);
				return;
			}
			
			 // 5. 返回到图书列表
			request.getRequestDispatcher("/admin/AdminBookServlet?method=findAll")
					.forward(request, response);
		} catch (Exception e) { 
			if(e instanceof FileUploadBase.FileSizeLimitExceededException) {
				request.setAttribute("msg", "您上传的文件超出了15KB");
				request.setAttribute("categoryList", categoryService.findAll());
				request.getRequestDispatcher("/adminjsps/admin/book/add.jsp")
						.forward(request, response);
			}
		}
	}
}
```
