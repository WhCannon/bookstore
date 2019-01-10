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

```

## OrderService
```java

```

## OrderServlet
```java

```
