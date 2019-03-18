# 订单模块
  - [Order](#order)
  - [OrderItem](#orderitem)
  - [OrderDao](#orderdao)
  - [OrderService](#orderservice)
  - [OrderServlet](#orderservlet)
  
  

## Order
```java
public class Order {
	private String oid;
	private Date ordertime;// 下单时间
	private double total;// 合计
	private int state;// 订单状态有四种：1未付款 2已付款但未发货 3已发货但未确认收货 4已确认交易成功
	private User owner;// 订单所有者
	private String address;// 收货地址
	private List<OrderItem> orderItemList;//当前订单下所有条目
  	getters和setters
  }
```

## OrderItem
```java
// 订单条目类
public class OrderItem {
	private String iid;
	private int count;// 数量
	private double subtotal;// 小计
	private Order order;// 所属订单
	private Book book;// 所要购买的图书
  	getters和setters
  }
```

## OrderDao
```java
public class OrderDao {
	private QueryRunner qr = new TxQueryRunner();
	 
	// 添加订单
	public void addOrder(Order order) {
		try {
			String sql = "insert into orders values(?,?,?,?,?,?)";
			 // 这里需要把util的Date类型转换成sql的Timestamp类型
			Timestamp timestamp = new Timestamp(order.getOrdertime().getTime());
			Object[] params = {order.getOid(), timestamp, order.getTotal(),
					order.getState(), order.getOwner().getUid(),
					order.getAddress()};
			qr.update(sql, params);
		} catch(SQLException e) {
			throw new RuntimeException(e);
		}
	}
	
	 // 插入订单条目,进行批处理操作
	public void addOrderItemList(List<OrderItem> orderItemList) {
		/* QueryRunner类的batch(String sql, Object[][] params)
		 * 其中params是多个一维数组！
		 * 每个一维数组都与sql在一起执行一次，多个一维数组就执行多次 */
		try {
			String sql = "insert into orderitem values(?,?,?,?,?)";
			/* 把orderItemList转换成二维数组
			 * 把一个OrderItem对象转换成一个一维数组 */
			Object[][] params = new Object[orderItemList.size()][];
			// 循环遍历orderItemList，使用每个orderItem对象为params中每个一维数组赋值
			for(int i = 0; i < orderItemList.size(); i++) {
				OrderItem item = orderItemList.get(i);
				params[i] = new Object[]{item.getIid(), item.getCount(), 
						item.getSubtotal(), item.getOrder().getOid(),
						item.getBook().getBid()}; 
			}
			qr.batch(sql, params);//执行批处理
		} catch(SQLException e) {
			throw new RuntimeException(e);
		}
	}

	 // 按uid查询订单
	public List<Order> findByUid(String uid) {
		/* 1. 通过uid查询出当前用户的所有List<Order>
		 * 2. 循环遍历每个Order，为其加载他的所有OrderItem */
		try {
			 // 1. 得到当前用户的所有订单
			String sql = "select * from orders where uid=?";
			List<Order> orderList = qr.query(sql, new BeanListHandler<Order>(Order.class), uid);
			 // 2. 循环遍历每个Order，为其加载它自己所有的订单条目
			for(Order order : orderList) {
				loadOrderItems(order);//为order对象添加它的所有订单条目
			}
			 // 3. 返回订单列表
			return orderList;
		} catch(SQLException e) {
			throw new RuntimeException(e);
		}
	}

	 // 给指定的订单加载所有的订单条目
	private void loadOrderItems(Order order) throws SQLException {
		 // 查询两张表：orderitem、book
		String sql = "select * from orderitem i, book b where i.bid=b.bid and oid=?";
		 // 因为一行结果集对应的不再是一个javabean，所以不能再使用BeanListHandler，而是MapListHandler
		List<Map<String,Object>> mapList = qr.query(sql, new MapListHandler(), order.getOid());
		/* mapList是一个List,里面放了多个map，每个map对应一行结果集
		 * 一行：
		 * {iid=C7AD5492F27D492189105FB50E55CBB6, count=2, 
		 subtotal=60.0, oid=1AE8A70354C947F8B81B80ADA6783155, bid=7, 
		 bname=精通Hibernate,price=30.0, author=张卫琴, image=book_img/8991366-1_l.jpg, cid=2}
		 * 我们需要使用一个Map生成两个对象：OrderItem、Book，然后再建立两者的关系（把Book设置给OrderItem） */
		
		 // 循环遍历每个Map，使用map生成两个对象，然后建立关系（最终得到一个OrderItem），把OrderItem保存起来
		List<OrderItem> orderItemList = toOrderItemList(mapList);
		order.setOrderItemList(orderItemList);
	}

	 // 把mapList中每个Map转换成两个对象，并建立关系
	private List<OrderItem> toOrderItemList(List<Map<String, Object>> mapList) {
		List<OrderItem> orderItemList = new ArrayList<OrderItem>();
		for(Map<String,Object> map : mapList) {
			OrderItem item = toOrderItem(map);
			orderItemList.add(item);
		}
		return orderItemList;
	}

	 // 把一个Map转换成一个OrderItem对象
	private OrderItem toOrderItem(Map<String, Object> map) {
		OrderItem orderItem = CommonUtils.toBean(map, OrderItem.class);
		Book book = CommonUtils.toBean(map, Book.class);
		orderItem.setBook(book);
		return orderItem;
	}

	 // 加载订单
	public Order load(String oid) {
		try {
			 // 1. 得到当前订单
			String sql = "select * from orders where oid=?";
			Order order = qr.query(sql, new BeanHandler<Order>(Order.class), oid);
			
			 // 2. 为order加载它的所有条目
			loadOrderItems(order);
			 
			// 3. 返回订单
			return order;
		} catch(SQLException e) {
			throw new RuntimeException(e);
		}
	}
	
	 //  通过oid查询订单状态
	public int getStateByOid(String oid) {
		try {
			String sql = "select state from orders where oid=?";
			return (Integer)qr.query(sql, new ScalarHandler(), oid);
		} catch(SQLException e) {
			throw new RuntimeException(e);
		}
	}
	
	 // 修改订单状态
	public void updateState(String oid, int state) {
		try {
			String sql = "update orders set state=? where oid=?";
			qr.update(sql, state, oid);
		} catch(SQLException e) {
			throw new RuntimeException(e);
		}
	}
}
```

## OrderService
```java
public class OrderService {
	private OrderDao orderDao = new OrderDao();
	
	// 支付方法
	public void zhiFu(String oid) {
		/* 1. 获取订单的状态
		 *    如果状态为1，那么执行下面代码
		 *    如果状态不为1，那么本方法什么都不做 */
		int state = orderDao.getStateByOid(oid);
		if(state == 1) {
			// 修改订单状态为2
			orderDao.updateState(oid, 2);
		}
	}
	
	 // 添加订单,需要处理事务
	public void add(Order order) {
		try {
			// 开启事务
			JdbcUtils.beginTransaction();
			orderDao.addOrder(order);//插入订单
			orderDao.addOrderItemList(order.getOrderItemList());//插入订单中的所有条目
			// 提交事务
			JdbcUtils.commitTransaction();
		} catch(Exception e) {
			// 回滚事务
			try {
				JdbcUtils.rollbackTransaction();
			} catch (SQLException e1) {
			}
			throw new RuntimeException(e);
		}
	}

	 // 我的订单
	public List<Order> myOrders(String uid) {
		return orderDao.findByUid(uid);
	}

	 // 加载订单
	public Order load(String oid) {
		return orderDao.load(oid);
	}
	
	 // 确认收货
	public void confirm(String oid) throws OrderException {
		 // 1. 校验订单状态，如果不是3，抛出异常
		int state = orderDao.getStateByOid(oid);//获取订单状态
		if(state != 3) throw new OrderException("订单确认失败，您不是什么好东西！");
		
		 // 2. 修改订单状态为4，表示交易成功
		orderDao.updateState(oid, 4);
	}
}

```

## OrderServlet
```java
public class OrderServlet extends BaseServlet {
	private OrderService orderService = new OrderService();
	
	// 支付之去银行
	public String zhiFu(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		Properties props = new Properties();
		InputStream input = this.getClass().getClassLoader()
				.getResourceAsStream("merchantInfo.properties");
		props.load(input);
		 // 准备13参数
		String p0_Cmd = "Buy";
		String p1_MerId = props.getProperty("p1_MerId");
		String p2_Order = request.getParameter("oid");
		String p3_Amt = "0.01";
		String p4_Cur = "CNY";
		String p5_Pid = "";
		String p6_Pcat = "";
		String p7_Pdesc = "";
		String p8_Url = props.getProperty("p8_Url");
		String p9_SAF = "";
		String pa_MP = "";
		String pd_FrpId = request.getParameter("pd_FrpId");
		String pr_NeedResponse = "1";

		 // 计算hmac
		String keyValue = props.getProperty("keyValue");
		String hmac = PaymentUtil.buildHmac(p0_Cmd, p1_MerId, p2_Order, p3_Amt,
				p4_Cur, p5_Pid, p6_Pcat, p7_Pdesc, p8_Url, p9_SAF, pa_MP,
				pd_FrpId, pr_NeedResponse, keyValue);

		 // 连接易宝的网址和13+1个参数
		StringBuilder url = new StringBuilder(props.getProperty("url"));
		url.append("?p0_Cmd=").append(p0_Cmd);
		url.append("&p1_MerId=").append(p1_MerId);
		url.append("&p2_Order=").append(p2_Order);
		url.append("&p3_Amt=").append(p3_Amt);
		url.append("&p4_Cur=").append(p4_Cur);
		url.append("&p5_Pid=").append(p5_Pid);
		url.append("&p6_Pcat=").append(p6_Pcat);
		url.append("&p7_Pdesc=").append(p7_Pdesc);
		url.append("&p8_Url=").append(p8_Url);
		url.append("&p9_SAF=").append(p9_SAF);
		url.append("&pa_MP=").append(pa_MP);
		url.append("&pd_FrpId=").append(pd_FrpId);
		url.append("&pr_NeedResponse=").append(pr_NeedResponse);
		url.append("&hmac=").append(hmac);
		System.out.println(url);

		 //重定向到易宝
		response.sendRedirect(url.toString());
		return null;
	}
	
	 // 这个方法是易宝回调方法,我们必须要判断调用本方法的是不是易宝！
	public String back(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		// 1. 获取11 + 1
		String p1_MerId = request.getParameter("p1_MerId");
		String r0_Cmd = request.getParameter("r0_Cmd");
		String r1_Code = request.getParameter("r1_Code");
		String r2_TrxId = request.getParameter("r2_TrxId");
		String r3_Amt = request.getParameter("r3_Amt");
		String r4_Cur = request.getParameter("r4_Cur");
		String r5_Pid = request.getParameter("r5_Pid");
		String r6_Order = request.getParameter("r6_Order");
		String r7_Uid = request.getParameter("r7_Uid");
		String r8_MP = request.getParameter("r8_MP");
		String r9_BType = request.getParameter("r9_BType");

		String hmac = request.getParameter("hmac");

		 // 2. 校验访问者是否为易宝！
		Properties props = new Properties();
		InputStream input = this.getClass().getClassLoader()
				.getResourceAsStream("merchantInfo.properties");
		props.load(input);
		String keyValue = props.getProperty("keyValue");

		boolean bool = PaymentUtil.verifyCallback(hmac, p1_MerId, r0_Cmd,
				r1_Code, r2_TrxId, r3_Amt, r4_Cur, r5_Pid, r6_Order, r7_Uid,
				r8_MP, r9_BType, keyValue);
		
		if(!bool) {//如果校验失败
			request.setAttribute("msg", "您不是什么好东西！");
			return "f:/jsps/msg.jsp";
		}
		
		 // 3. 获取状态订单，确定是否要修改订单状态，以及添加积分等业务操作
		orderService.zhiFu(r6_Order);//有可能对数据库进行操作，也可能不操作！
		
		 // 4. 判断当前回调方式
		 //  如果为点对点，需要回馈以success开头的字符串
		if(r9_BType.equals("2")) {
			response.getWriter().print("success");
		}
		
		 // 5. 保存成功信息，转发到msg.jsp
		request.setAttribute("msg", "支付成功！等待卖家发货！你慢慢等~");
		return "f:/jsps/msg.jsp";
	}
	
	// 确认收货
	public String confirm(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		/* 1. 获取oid参数
		 * 2. 调用service方法
		 *   > 如果有异常，保存异常信息，转发到msg.jsp
		 * 3. 保存成功信息，转发到msg.jsp */
		String oid = request.getParameter("oid");
		try {
			orderService.confirm(oid);
			request.setAttribute("msg", "恭喜，交易成功！");
		} catch (OrderException e) {
			request.setAttribute("msg", e.getMessage());
		}
		return "f:/jsps/msg.jsp";
	}
	
	 // 加载订单
	public String load(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		/* 1. 得到oid参数
		 * 2. 使用oid调用service方法得到Order
		 * 3. 保存到request域，转发到/jsps/order/desc.jsp */
		request.setAttribute("order", orderService.load(request.getParameter("oid")));
		return "f:/jsps/order/desc.jsp";
	}
	
	 //我的订单
	public String myOrders(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		/* 1. 从session得到当前用户，再获取其uid
		 * 2. 使用uid调用orderService#myOrders(uid)得到该用户的所有订单List<Order>
		 * 3. 把订单列表保存到request域中，转发到/jsps/order/list.jsp */
		User user = (User)request.getSession().getAttribute("session_user");
		List<Order> orderList = orderService.myOrders(user.getUid());
		request.setAttribute("orderList", orderList);
		return "f:/jsps/order/list.jsp";
	}
	
	 // 添加订单，把session中的车用来生成Order对象
	public String add(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {
		/* 1. 从session中得到cart
		 * 2. 使用cart生成Order对象
		 * 3. 调用service方法完成添加订单
		 * 4. 保存order到request域中，转发到/jsps/order/desc.jsp */
		
		// 从session中获取cart
		Cart cart = (Cart)request.getSession().getAttribute("cart");
		
		// 把cart转换成Order对象
		// 创建Order对象，并设置属性
		Order order = new Order();
		order.setOid(CommonUtils.uuid());//设置编号
		order.setOrdertime(new Date());//设置下单时间
		order.setState(1);//设置订单状态为1，表示未付款
		User user = (User)request.getSession().getAttribute("session_user");
		order.setOwner(user);//设置订单所有者
		order.setTotal(cart.getTotal());//设置订单的合计，从cart中获取合计
		
		 // 创建订单条目集合
		List<OrderItem> orderItemList = new ArrayList<OrderItem>();
		// 循环遍历Cart中的所有CartItem，使用每一个CartItem对象创建OrderItem对象，并添加到集合中
		for(CartItem cartItem : cart.getCartItems()) {
			OrderItem oi = new OrderItem();//创建订单条目
			
			oi.setIid(CommonUtils.uuid());//设置条目的id
			oi.setCount(cartItem.getCount());//设置条目的数量
			oi.setBook(cartItem.getBook());//设置条目的图书
			oi.setSubtotal(cartItem.getSubtotal());//设置条目的小计
			oi.setOrder(order);//设置所属订单
			
			orderItemList.add(oi);//把订单条目添加到集合中
		}
		
		// 把所有的订单条目添加到订单中
		order.setOrderItemList(orderItemList);
		
		// 清空购物车
		cart.clear();
		
		 // 调用orderService添加订单
		orderService.add(order);
		
		 // 保存order到request域，转发到/jsps/order/desc.jsp
		request.setAttribute("order", order);
		return "/jsps/order/desc.jsp";
	}
}
```
