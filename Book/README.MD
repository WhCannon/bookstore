# 书模块
  - [domain](#book)
  - [servlet](#bookservlet)
  - [service](#bookservice)
  - [dao](#bookdao)
  

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
			String sql = "select * from book";
			return qr.query(sql, new BeanListHandler<Book>(Book.class));
		} catch(SQLException e) {
			throw new RuntimeException(e);
		}
	}

	 // 按分类查询
	public List<Book> findByCategory(String cid) {
		try {
			String sql = "select * from book where cid=?";
			return qr.query(sql, new BeanListHandler<Book>(Book.class), cid);
		} catch(SQLException e) {
			throw new RuntimeException(e);
		}
	}

	 // 查看图书详细信息
	public Book findByBid(String bid) {
		try {
			String sql = "select * from book where bid=?";
			return qr.query(sql, new BeanHandler<Book>(Book.class), bid);
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

	// 查看图书信息
	public Book load(String bid) {
		return bookDao.findByBid(bid);
	}
}
```

## BookServlet
```
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
