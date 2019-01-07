# 购物车模块
  - [Cart](#cart)
  - [CartItem](#cartitem)
  - [Servlet](#cartservlet)
  


## Cart
```java
// 购物车类
public class Cart {
	private Map<String,CartItem> map = new LinkedHashMap<String,CartItem>();
	 // 计算合计
	public double getTotal() {
		// 合计=所有条目的小计之和
		BigDecimal total = new BigDecimal("0");
		for(CartItem cartItem : map.values()) {
			BigDecimal subtotal = new BigDecimal("" + cartItem.getSubtotal());
			total = total.add(subtotal);
		}
		return total.doubleValue();
	}
	
	 // 添加条目到车中
	public void add(CartItem cartItem) {
		if(map.containsKey(cartItem.getBook().getBid())) {//判断原来车中是否存在该条目
			CartItem _cartItem = map.get(cartItem.getBook().getBid());//返回原条目
			_cartItem.setCount(_cartItem.getCount() + cartItem.getCount());//设置老条目的数量为，其自己数量+新条目的数量
			map.put(cartItem.getBook().getBid(), _cartItem);
		} else {
			map.put(cartItem.getBook().getBid(), cartItem);
		}
	}
	
	 // 清空所有条目
	public void clear() {
		map.clear();
	}
	
	 // 删除指定条目
	public void delete(String bid) {
		map.remove(bid);
	}
	
	 // 获取所有条目
	public Collection<CartItem> getCartItems() {
		return map.values();
	}
}
```
## CartItem
```java
// 购物车条目类
public class CartItem {
	private Book book;// 商品
	private int count;// 数量
	
	 // 小计方法,处理了二进制运算误差问题
	public double getSubtotal() {//小计方法，但它没有对应的成员！
		BigDecimal d1 = new BigDecimal(book.getPrice() + "");
		BigDecimal d2 = new BigDecimal(count + "");
		return d1.multiply(d2).doubleValue();
	}

	public Book getBook() {
		return book;
	}

	public void setBook(Book book) {
		this.book = book;
	}

	public int getCount() {
		return count;
	}

	public void setCount(int count) {
		this.count = count;
	}
}
```

## CartServlet
```java

```
  

