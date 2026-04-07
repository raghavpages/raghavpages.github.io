---
layout: post
title: "State transfer on the web"
date: 2026-04-07 00:51:10
comments: true
description: "State transfer on the web"
keywords: ""
categories:

tags:

---
When I started working on the web, quite often it was building only web applications. Not static HTML pages. It involved presenting data sent by the server, user performing some action and then updating the data. Many times the same data had to be available across multiple pages along with the updated data before the user completed their journey by doing a last stage submit. Even though today's client side frameworks try to make this look a lot simpler, there are some techniques that were used before their development which might be applicable even today.

Let's take the example of an E-commerce site which has use cases ranging from displaying products on the page to adding items to cart, checking them out and even creating an account.Each of these use cases have specific asks of state transfer between pages.

## Server side code
Let's wind the clock to a time when server side rendered pages dominated and developers were still trying to figure out how to use the Javascript running in the browser. The entire HTML and layout of the page was rendered on the server side and returned back as a static HTML page to the browser. Technologies such as PHP or JSP (Java server pages) come to mind. All the dynamism related to the page was handled by the server.

### Query param
Take the use case of displaying products the e-commerce site wants to sell. In a store with a large inventory, there are lot of products even in each category. A common practice is to display them as a grid and spread them across multiple pages. The number of pages would be according to how many items are shown in each page. The navigation to the next page is usually at the bottom after scrolling through all the items on the current page. Typically, there is a hyperlink with a text "Next" or an arrow icon to navigate to the next page. This is how it is usually configured:
{% highlight html %}
<a href="/products?page=2">Next</a>
{% endhighlight %}
Once you navigate to the next page, there will also be a link to the previous page which can be calculated from the current page number on the server side. In JSP, this can be done using the following code:
{% highlight jsp %}
<a href="/products?page=<%= request.getParameter("page") - 1 %>">Previous</a>
{% endhighlight %}
The dynamic data is rendered using a placeholder syntax like `<%= ... %>` in the example above. So, to keep track of what page the user is on, a query param such as `page` is used which is read by the server to fetch the next set of products from a persistence layer. This will be visible in the browser's address bar which might catch the attention of a keen observer. 

### Hidden input
While query params worked well for the use case of pagination, it quickly starts getting harder if we have to keep track of more things using just query parameters. Take the example of viewing the cart that has the list of items checked out. You can adjust the quantity of each item one last time before checking out. Once you click checkout, you go to the payment page that has a form to enter you payment and shipping information. Because there is a transition between pages, the data from the cart page needs to be transferred to the checkout page. A simple way for this data to be made available on the next page is by creating hidden inputs in the form on the cart page so that while placing the order, we can also include the items that are part of the order. 

Here is how the JSP code might look to generate those hidden inputs from a cart:
{% highlight jsp %}
<% 
  // Loop through cart items and create hidden input fields for each
  List<CartItem> cartItems = (List<CartItem>) request.getAttribute("cartItems");
  for (int i = 0; i < cartItems.size(); i++) { 
%>
  <input type="hidden" name="productId_<%= i %>" value="<%= cartItems.get(i).getId() %>" />
  <input type="hidden" name="quantity_<%= i %>" value="<%= cartItems.get(i).getQuantity() %>" />
  <input type="hidden" name="price_<%= i %>" value="<%= cartItems.get(i).getPrice() %>" />
<% } %>
{% endhighlight %}
Now, when the order is placed and the order is created, all this information will be included in the order and be helpful in keeping track of what items to be shipped.

### Cookies and cache
This section may sound more like an ice-cream flavor. But it is "cookies and cache". There's no cream in it. Take the use case of adding items to a cart but you are not ready to make a purchase and want to return later. When you come back, it is hard if you want to again search for those items and add them to the cart when you are ready to make a purchase. But, there is an option available in the browser that can solve this problem. As and when you add items to the cart, the server can update a cookie in the browser. These are server side cookies. Cookies are just key value pairs that are part of every request and response headers of a HTTP request. The browser stores the cookies when it receives the response headers into the device's storage where the browser is installed. Typically these cookies are tied to the domain where the e-commerce site is hosted like www.example.com. More precisely, partitioned by origin. This is how cookies are sent in the response header:
{% highlight html %}
Set-Cookie: cartId=1234567890; Path=/; HttpOnly; Secure
{% endhighlight %}
In any request made containing this domain, the browser will send these stored cookies in the request headers. The server will be able to read these cookies and update the cart accordingly. This is how cookies are sent in the request header:
{% highlight html %}
Cookie: cartId=1234567890
{% endhighlight %}
In a different use case when a user logs in, a session for the user is created. This is stored as an identifier in the cookies. On the server, there is a cache which has a reference to this identifier in the cookies. After user logs in, their profile information could be pulled and stored in a cache. So, even if they navigate through multiple pages shopping around, at the time they checkout, their address would be prefilled. It is retrieved from the cache. The cookies itself can be configured to stay on while the browser is open or for a longer duration even after the browser is closed and reopened. For this, the cookie is set with an expiry in the response headers:
{% highlight html %}
Set-Cookie: sessionId=1234567890; Path=/; HttpOnly; Secure; Expires=Wed, 21 Oct 2025 07:28:00 GMT
{% endhighlight %}

## Client side code
In a more modern web application, there is enough code running in the browser taking on more responsiblity. The real game changer was Javascript code firing requests to the server through XMLHttpRequest (XHR) or AJAX. In a more recent development, the fetch API does this job more elegantly. Javascript also has APIs to do DOM (document object model) manipulation. This can be used to simulate page transitions instead of loading entire pages from the server. 

### Initial data
The client side code still needs some initial data to be fed to it to set the ball rolling in the browser. This can contain the data to run some display logic according to business rules and affect the user experience.

#### Data attribute
This technique was quite popular in the days of JQuery which dominated as a Javascript framework for the client side. The properties that the client side code needs immediately on page load is added as attributes mainly to the body tag of the HTML. These attributes usually have a prefix of `data-` before the property name. This is how it would be added to the page:
{% highlight html %}
<body data-user-id="1234567890" data-user-name="John Doe">
{% endhighlight %}
Using JQuery syntax, this is how the property can be accessed:
{% highlight javascript %}
var userId = $("body").data("user-id");
var userName = $("body").data("user-name");
{% endhighlight %}
Other elements can also be used to store data attributes. If for some reason the data attribute needs to be updated, the same method can be used to update the data attributes. For example, when updating items to a cart and showing the count as a bubble next to the cart icon. This is how the count could be updated and kept track:
{% highlight javascript %}
var currentCartCount = $("div#cart-icon").data("cart-count");
$("div#cart-icon").data("cart-count", currentCartCount + 1);
{% endhighlight %}

#### JSON in script tag
The approach is used quite often in modern frameworks like React, Angular, and Vue.js. The data is embedded in a script tag as a JSON string. This is how it would look like:
{% highlight html %}
<script id="initial-data" type="application/json">
  "{\"userId\":\"1234567890\",\"userName\":\"John Doe\"}"
</script>
{% endhighlight %}
This is then parsed by first accessing the inner HTML of the script tag element and fed into client side state management libraries like Redux, Apollo cache, React context etc. While data attributes worked well to access individual properties, with this approach, multiple properties can be collected into a single object and be part of a global state. In React, a context provider component is wrapped on top of the component tree and the state is made available to all components in the tree. This is how a native React Context provider is set up:
{% highlight javascript %}
const AppContext = React.createContext();

<AppContext.Provider value={JSON.parse(document.getElementById("initial-data").innerHTML)}>
  <App />
</AppContext.Provider>
{% endhighlight %}
This allows any component within the `App` access to the initial data from the server enough to paint a UI to the user.

### Client side storage
The browser offers storage options for client side code without the dependency on cookies or calls to the server. Just like in the case of cookies, the types of storage options available depend on whether the data is needed only while the browser is open or even after the browser is closed and reopened. Another similarity they share with cookies is that they are tied to the domain of the page that is requested or partitioned by origin. The two options are `sessionStorage` and `localStorage` offered by the storage API in the browser. It is easy to guess that `sessionStorage` allows us to store data that lasts only while the page is open. While, `localStorage` is for use cases that need the data even after the page is closed and reopened. The data again is stored as key value pairs. They have getter and setter methods to read and write to specific keys in the browser storage. In the E-commerce site example, `localStorage` could be used to save the user's cart in the browser. While trying to preserve state when user clicks the back button, `sessionStorage` could be helpful to retrieve what the user was doing until that point. These are the main storage options available on the browser. There are other options like IndexedDB for more complex data storage needs.

We just covered a specific but an important topic in front end development for the web. Being stateless has its advantages while the web and HTTP was meant to serve static pages. Due to its growing popularity, it became useful to monetize and paved the way for sites to sell consumer products like the E-commerce example discussed in this article. In order to scale to a growing list of products, sellers and users, apart from building the infrastructure, the pages served to the users needed to be dynamically generated and the data moving between them needed to be tracked. So, the concept of state had to be implemented for the web. This is an attempt at going over a somewhat exhaustive list of state transfer options while still maintaining the fundamental ideas that each option brings.