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
		/* mapList是多个map，每个map对应一行结果集
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

```
