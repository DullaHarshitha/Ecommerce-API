# Ecommerce-API
import java.util.*;
import java.util.stream.Collectors;

// Simple in-memory user store and authentication
class User {
    private String username;
    private String password;
    private String role; // "customer" or "admin"

    public User(String username, String password, String role) {
        this.username = username;
        this.password = password;
        this.role = role;
    }
    public String getUsername() { return username; }
    public String getPassword() { return password; }
    public String getRole() { return role; }
}

class Product {
    private static int COUNTER = 1;
    private int id;
    private String name;
    private String category;
    private double price;

    public Product(String name, String category, double price) {
        this.id = COUNTER++;
        this.name = name;
        this.category = category;
        this.price = price;
    }
    public int getId() { return id; }
    public String getName() { return name; }
    public String getCategory() { return category; }
    public double getPrice() { return price; }
    public void setName(String name) { this.name = name; }
    public void setCategory(String category) { this.category = category; }
    public void setPrice(double price) { this.price = price; }
    public String toString() {
        return String.format("Product{id=%d, name='%s', category='%s', price=%.2f}", id, name, category, price);
    }
}

class CartItem {
    private Product product;
    private int quantity;
    public CartItem(Product product, int quantity) {
        this.product = product;
        this.quantity = quantity;
    }
    public Product getProduct() { return product; }
    public int getQuantity() { return quantity; }
    public void setQuantity(int quantity) { this.quantity = quantity; }
    public String toString() {
        return product.toString() + " x" + quantity;
    }
}

class Order {
    private static int COUNTER = 1;
    private int id;
    private List<CartItem> cartItems;
    private double total;
    private String status;
    public Order(List<CartItem> cartItems) {
        this.id = COUNTER++;
        this.cartItems = new ArrayList<>(cartItems);
        this.total = 0.0;
        for (CartItem item : cartItems) {
            this.total += item.getProduct().getPrice() * item.getQuantity();
        }
        this.status = "PENDING";
    }
    public int getId() { return id; }
    public List<CartItem> getCartItems() { return cartItems; }
    public double getTotal() { return total; }
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
    public String toString() {
        StringBuilder sb = new StringBuilder();
        sb.append("Order{id=").append(id).append(", items=[");
        for (int i = 0; i < cartItems.size(); i++) {
            sb.append(cartItems.get(i).toString());
            if (i < cartItems.size() - 1) sb.append(", ");
        }
        sb.append("], total=").append(String.format("%.2f", total)).append(", status=").append(status).append("}");
        return sb.toString();
    }
}

// Simulated JWT utility (for demo purposes only)
class JwtUtil {
    // In a real app, use a library to generate/validate JWTs securely
    public static String generateToken(String username, String role) {
        // Very basic: username:role:timestamp
        return Base64.getEncoder().encodeToString((username + ":" + role + ":" + System.currentTimeMillis()).getBytes());
    }
    public static String[] parseToken(String token) {
        try {
            String decoded = new String(Base64.getDecoder().decode(token));
            String[] parts = decoded.split(":");
            if (parts.length >= 2) {
                return new String[]{parts[0], parts[1]}; // username, role
            }
        } catch (Exception e) {}
        return null;
    }
}

class ECommerceService {
    private List<Product> products = new ArrayList<>();
    private Map<String, List<CartItem>> carts = new HashMap<>(); // username -> cart
    private List<Order> orders = new ArrayList<>();
    private List<User> users = new ArrayList<>();

    public ECommerceService() {
        // Sample products
        products.add(new Product("Laptop", "Electronics", 1200.00));
        products.add(new Product("Phone", "Electronics", 800.00));
        products.add(new Product("Shoes", "Apparel", 120.00));
        // Sample users
        users.add(new User("alice", "password", "customer"));
        users.add(new User("bob", "adminpass", "admin"));
    }

    // Authentication
    public String login(String username, String password) {
        for (User u : users) {
            if (u.getUsername().equals(username) && u.getPassword().equals(password)) {
                return JwtUtil.generateToken(u.getUsername(), u.getRole());
            }
        }
        return null;
    }
    public User getUserFromToken(String token) {
        String[] parsed = JwtUtil.parseToken(token);
        if (parsed == null) return null;
        for (User u : users) {
            if (u.getUsername().equals(parsed[0]) && u.getRole().equals(parsed[1])) {
                return u;
            }
        }
        return null;
    }

    // Product Listing with optional pagination and search
    public List<Product> listProducts(int page, int pageSize, String search, String category) {
        List<Product> filtered = products;
        if (search != null && !search.isEmpty()) {
            filtered = filtered.stream()
                .filter(p -> p.getName().toLowerCase().contains(search.toLowerCase()))
                .collect(Collectors.toList());
        }
        if (category != null && !category.isEmpty()) {
            filtered = filtered.stream()
                .filter(p -> p.getCategory().equalsIgnoreCase(category))
                .collect(Collectors.toList());
        }
        int from = Math.max(0, (page - 1) * pageSize);
        int to = Math.min(filtered.size(), from + pageSize);
        if (from > to) return new ArrayList<>();
        return filtered.subList(from, to);
    }

    // Admin: Add, Update, Delete Product
    public boolean addProduct(Product p, User user) {
        if (!"admin".equals(user.getRole())) return false;
        products.add(p);
        return true;
    }
    public boolean updateProduct(int id, String name, String category, double price, User user) {
        if (!"admin".equals(user.getRole())) return false;
        for (Product p : products) {
            if (p.getId() == id) {
                p.setName(name);
                p.setCategory(category);
                p.setPrice(price);
                return true;
            }
        }
        return false;
    }
    public boolean deleteProduct(int id, User user) {
        if (!"admin".equals(user.getRole())) return false;
        return products.removeIf(p -> p.getId() == id);
    }

    // Cart Management
    public List<CartItem> getCart(String username) {
        return carts.getOrDefault(username, new ArrayList<>());
    }
    public void addToCart(String username, int productId, int quantity) {
        List<CartItem> cart = carts.getOrDefault(username, new ArrayList<>());
        for (CartItem item : cart) {
            if (item.getProduct().getId() == productId) {
                item.setQuantity(item.getQuantity() + quantity);
                carts.put(username, cart);
                return;
            }
        }
        for (Product p : products) {
            if (p.getId() == productId) {
                cart.add(new CartItem(p, quantity));
                break;
            }
        }
        carts.put(username, cart);
    }
    public void updateCartItem(String username, int productId, int quantity) {
        List<CartItem> cart = carts.getOrDefault(username, new ArrayList<>());
        for (CartItem item : cart) {
            if (item.getProduct().getId() == productId) {
                item.setQuantity(quantity);
                break;
            }
        }
        carts.put(username, cart);
    }
    public void removeCartItem(String username, int productId) {
        List<CartItem> cart = carts.getOrDefault(username, new ArrayList<>());
        cart.removeIf(item -> item.getProduct().getId() == productId);
        carts.put(username, cart);
    }
    public void clearCart(String username) {
        carts.remove(username);
    }

    // Order Creation
    public Order createOrder(String username) {
        List<CartItem> cart = carts.getOrDefault(username, new ArrayList<>());
        if (cart.isEmpty()) return null;
        Order order = new Order(cart);
        orders.add(order);
        clearCart(username);
        return order;
    }
    public List<Order> getOrders(String username, String role) {
        if ("admin".equals(role)) {
            return orders;
        } else {
            // For demo, assume all orders are by this user (no user field in Order)
            return orders;
        }
    }
}

// Simple CLI for demonstration
class ECommerceCLI {
    private static Scanner scanner = new Scanner(System.in);
    private static ECommerceService service = new ECommerceService();
    private static String token = null;
    private static User currentUser = null;

    public static void main(String[] args) {
        System.out.println("Welcome to E-Commerce API (CLI Demo)");
        while (true) {
            if (token == null) {
                System.out.print("Login (username): ");
                String username = scanner.nextLine();
                System.out.print("Password: ");
                String password = scanner.nextLine();
                token = service.login(username, password);
                if (token == null) {
                    System.out.println("Invalid credentials.");
                } else {
                    currentUser = service.getUserFromToken(token);
                    System.out.println("Login successful as " + currentUser.getRole());
                }
                continue;
            }
            System.out.println("\nMenu:");
            System.out.println("1. List Products");
            System.out.println("2. Search Products");
            System.out.println("3. View Cart");
            System.out.println("4. Add to Cart");
            System.out.println("5. Update Cart Item");
            System.out.println("6. Remove Cart Item");
            System.out.println("7. Create Order");
            System.out.println("8. View Orders");
            if ("admin".equals(currentUser.getRole())) {
                System.out.println("9. Add Product");
                System.out.println("10. Update Product");
                System.out.println("11. Delete Product");
            }
            System.out.println("0. Logout");
            System.out.print("Choose option: ");
            String opt = scanner.nextLine();
            try {
                switch (opt) {
                    case "1":
                        listProducts();
                        break;
                    case "2":
                        searchProducts();
                        break;
                    case "3":
                        viewCart();
                        break;
                    case "4":
                        addToCart();
                        break;
                    case "5":
                        updateCartItem();
                        break;
                    case "6":
                        removeCartItem();
                        break;
                    case "7":
                        createOrder();
                        break;
                    case "8":
                        viewOrders();
                        break;
                    case "9":
                        if ("admin".equals(currentUser.getRole())) addProduct();
                        break;
                    case "10":
                        if ("admin".equals(currentUser.getRole())) updateProduct();
                        break;
                    case "11":
                        if ("admin".equals(currentUser.getRole())) deleteProduct();
                        break;
                    case "0":
                        token = null;
                        currentUser = null;
                        System.out.println("Logged out.");
                        break;
                    default:
                        System.out.println("Invalid option.");
                }
            } catch (Exception e) {
                System.out.println("Error: " + e.getMessage());
            }
        }
    }

    private static void listProducts() {
        System.out.print("Page (default 1): ");
        String pageStr = scanner.nextLine();
        int page = pageStr.isEmpty() ? 1 : Integer.parseInt(pageStr);
        int pageSize = 5;
        List<Product> products = service.listProducts(page, pageSize, null, null);
        if (products.isEmpty()) {
            System.out.println("No products found.");
        } else {
            for (Product p : products) {
                System.out.println(p);
            }
        }
    }
    private static void searchProducts() {
        System.out.print("Search by name: ");
        String search = scanner.nextLine();
        System.out.print("Category (optional): ");
        String category = scanner.nextLine();
        List<Product> products = service.listProducts(1, 10, search, category);
        if (products.isEmpty()) {
            System.out.println("No products found.");
        } else {
            for (Product p : products) {
                System.out.println(p);
            }
        }
    }
    private static void viewCart() {
        List<CartItem> cart = service.getCart(currentUser.getUsername());
        if (cart.isEmpty()) {
            System.out.println("Cart is empty.");
        } else {
            for (CartItem item : cart) {
                System.out.println(item);
            }
        }
    }
    private static void addToCart() {
        System.out.print("Product ID: ");
        int pid = Integer.parseInt(scanner.nextLine());
        System.out.print("Quantity: ");
        int qty = Integer.parseInt(scanner.nextLine());
        service.addToCart(currentUser.getUsername(), pid, qty);
        System.out.println("Added to cart.");
    }
    private static void updateCartItem() {
        System.out.print("Product ID: ");
        int pid = Integer.parseInt(scanner.nextLine());
        System.out.print("New Quantity: ");
        int qty = Integer.parseInt(scanner.nextLine());
        service.updateCartItem(currentUser.getUsername(), pid, qty);
        System.out.println("Cart item updated.");
    }
    private static void removeCartItem() {
        System.out.print("Product ID: ");
        int pid = Integer.parseInt(scanner.nextLine());
        service.removeCartItem(currentUser.getUsername(), pid);
        System.out.println("Cart item removed.");
    }
    private static void createOrder() {
        Order order = service.createOrder(currentUser.getUsername());
        if (order == null) {
            System.out.println("Cart is empty. Cannot create order.");
        } else {
            System.out.println("Order created: " + order);
        }
    }
    private static void viewOrders() {
        List<Order> orders = service.getOrders(currentUser.getUsername(), currentUser.getRole());
        if (orders.isEmpty()) {
            System.out.println("No orders found.");
        } else {
            for (Order o : orders) {
                System.out.println(o);
            }
        }
    }
    private static void addProduct() {
        System.out.print("Name: ");
        String name = scanner.nextLine();
        System.out.print("Category: ");
        String category = scanner.nextLine();
        System.out.print("Price: ");
        double price = Double.parseDouble(scanner.nextLine());
        Product p = new Product(name, category, price);
        if (service.addProduct(p, currentUser)) {
            System.out.println("Product added.");
        } else {
            System.out.println("Failed to add product.");
        }
    }
    private static void updateProduct() {
        System.out.print("Product ID: ");
        int id = Integer.parseInt(scanner.nextLine());
        System.out.print("New Name: ");
        String name = scanner.nextLine();
        System.out.print("New Category: ");
        String category = scanner.nextLine();
        System.out.print("New Price: ");
        double price = Double.parseDouble(scanner.nextLine());
        if (service.updateProduct(id, name, category, price, currentUser)) {
            System.out.println("Product updated.");
        } else {
            System.out.println("Failed to update product.");
        }
    }
    private static void deleteProduct() {
        System.out.print("Product ID: ");
        int id = Integer.parseInt(scanner.nextLine());
        if (service.deleteProduct(id, currentUser)) {
            System.out.println("Product deleted.");
        } else {
            System.out.println("Failed to delete product.");
        }
    }
}
