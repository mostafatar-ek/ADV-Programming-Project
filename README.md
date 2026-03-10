# CampusConnect — Step-by-step Implementation Guide (Java Servlets + JDBC + MySQL)

> Current date reference: **2026-03-10**.
>
> This README is intentionally **very specific** and written as a checklist you can follow from zero → finished project.
>
> **Project idea:** CampusConnect is a campus marketplace for academic help: students create **offers/requests**, browse by **categories**, and interact via **requests** (and optionally messages). Admins can manage users, posts, and categories.

---

## 0) What you must have installed (exact)

1. **JDK 17** (or JDK 11+). Verify:
   - Run: `java -version`
   - You must see something like `17.x`.
2. **Apache Tomcat 9** (recommended for `javax.servlet.*`) or Tomcat 10 (only if you change to `jakarta.servlet.*`).
   - Recommendation: **Tomcat 9** to keep `javax.servlet`.
3. **MySQL 8**
4. **MySQL Workbench** (optional but very helpful)
5. IDE: **IntelliJ IDEA Ultimate** OR **Eclipse Enterprise** OR **NetBeans**
6. Git (optional)

---

## 1) Create the project (Dynamic Web App structure)

### Option A (recommended): IntelliJ IDEA + Tomcat

1. `File` → `New` → `Project` → choose **Java**.
2. Enable **Web Application** (Servlet).
3. Set:
   - GroupId: `com.campusconnect`
   - ArtifactId: `CampusConnect`
   - Packaging: `war`
4. Make sure the output is a **WAR**.
5. Add a Tomcat run configuration:
   - `Run` → `Edit Configurations` → `+` → `Tomcat Server` → `Local`
   - Deploy: add artifact `CampusConnect:war exploded`
   - Context path: `/CampusConnect`

### Option B: Eclipse (Dynamic Web Project)

1. `File` → `New` → `Dynamic Web Project`
2. Project name: `CampusConnect`
3. Target runtime: `Apache Tomcat v9.0`
4. Dynamic web module version: `3.1` (or `4.0`)
5. Finish

---

## 2) Final folder structure (follow exactly)

Use this structure (you listed it; implement it literally):

```
CampusConnect/
├── src/
│   ├── controller/
│   ├── dao/
│   ├── model/
│   └── util/
├── webapp/
│   ├── css/
│   ├── js/
│   ├── images/
│   ├── WEB-INF/
│   │   └── web.xml
│   ├── login.jsp
│   ├── register.jsp
│   ├── home.jsp
│   ├── profile.jsp
│   └── ...
├── database/
│   └── campusconnect.sql
└── README.md
```

**Important rule:**
- All Java classes go into `src/...`
- All JSP pages go into `webapp/...`
- `WEB-INF/web.xml` contains your servlet mappings **OR** you use `@WebServlet` annotations (either is ok, but be consistent).

---

## 3) Add required libraries (JDBC + servlet)

### If you use Maven (recommended)

Create `pom.xml` and add:

- MySQL driver (`mysql-connector-j`)
- Servlet API provided by Tomcat

If you are not using Maven:
- Download MySQL connector jar and add to your project libraries.

---

## 4) Create the database (MySQL) — do this first

### 4.1 Create database

1. Open MySQL Workbench.
2. Connect to your local MySQL.
3. Create schema:
   - Schema name: `campusconnect`
4. Click `Apply`.

### 4.2 Create tables

Create a file: `database/campusconnect.sql` and run it.

**Minimum tables** (based on your modules):

- `users`
- `categories`
- `posts`
- `requests`
- `messages` (optional if you want messaging now)

### 4.3 Recommended table columns (very specific)

#### users
- `user_id` INT PRIMARY KEY AUTO_INCREMENT
- `full_name` VARCHAR(100) NOT NULL
- `email` VARCHAR(150) NOT NULL UNIQUE
- `password_hash` VARCHAR(255) NOT NULL
- `phone` VARCHAR(30)
- `role` ENUM('student','admin') NOT NULL DEFAULT 'student'
- `created_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP

#### categories
- `category_id` INT PRIMARY KEY AUTO_INCREMENT
- `category_name` VARCHAR(80) NOT NULL UNIQUE

#### posts
- `post_id` INT PRIMARY KEY AUTO_INCREMENT
- `title` VARCHAR(150) NOT NULL
- `description` TEXT NOT NULL
- `post_type` ENUM('offer','request') NOT NULL
- `contact_info` VARCHAR(200)
- `created_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP
- `user_id` INT NOT NULL
- `category_id` INT NOT NULL
- FOREIGN KEY (`user_id`) REFERENCES users(`user_id`) ON DELETE CASCADE
- FOREIGN KEY (`category_id`) REFERENCES categories(`category_id`) ON DELETE RESTRICT

#### requests
- `request_id` INT PRIMARY KEY AUTO_INCREMENT
- `sender_id` INT NOT NULL
- `post_id` INT NOT NULL
- `message` VARCHAR(500)
- `status` ENUM('pending','accepted','rejected') NOT NULL DEFAULT 'pending'
- `request_date` TIMESTAMP DEFAULT CURRENT_TIMESTAMP
- FOREIGN KEY (`sender_id`) REFERENCES users(`user_id`) ON DELETE CASCADE
- FOREIGN KEY (`post_id`) REFERENCES posts(`post_id`) ON DELETE CASCADE

#### messages (optional)
- `message_id` INT PRIMARY KEY AUTO_INCREMENT
- `sender_id` INT NOT NULL
- `receiver_id` INT NOT NULL
- `post_id` INT
- `content` VARCHAR(1000) NOT NULL
- `sent_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP
- FOREIGN KEY (`sender_id`) REFERENCES users(`user_id`) ON DELETE CASCADE
- FOREIGN KEY (`receiver_id`) REFERENCES users(`user_id`) ON DELETE CASCADE

### 4.4 Insert starter categories (seed data)

Insert:
- Tutoring
- Notes
- Used Books
- Team Projects
- Programming Help
- Design Help

---

## 5) Implement DBConnection (JDBC) — must work before everything else

Create `src/util/DBConnection.java`.

Checklist:
1. URL format: `jdbc:mysql://localhost:3306/campusconnect?useSSL=false&serverTimezone=UTC`
2. Username: your MySQL username
3. Password: your MySQL password
4. Test connection with a simple servlet or a `main` method (temporary).

**Rule:** never open one global Connection forever. In each DAO method:
- open connection
- prepare statement
- execute
- close resources

---

## 6) Create model classes (Plain Java Objects)

Create these in `src/model/` exactly:

- `User.java`
- `Category.java`
- `Post.java`
- `Request.java`
- `Message.java` (optional)

Fields (as you wrote):

- User: `int userId`, `String fullName`, `String email`, `String password`, `String phone`, `String role`
- Category: `int categoryId`, `String categoryName`
- Post: `int postId`, `String title`, `String description`, `String postType`, `String contactInfo`, `String createdAt`, `int userId`, `int categoryId`
- Request: `int requestId`, `int senderId`, `int postId`, `String message`, `String status`, `String requestDate`
- Message: `int messageId`, `int senderId`, `int receiverId`, `int postId`, `String content`, `String sentAt`

**Important:**
- In the database you should store `password_hash`, but in the Java object you can keep `password` as the raw input **only during registration/login**.

---

## 7) Create DAO classes (all SQL goes here)

Create in `src/dao/`:

- `UserDAO.java`
- `CategoryDAO.java`
- `PostDAO.java`
- `RequestDAO.java`
- `MessageDAO.java` (optional)

### 7.1 UserDAO — required queries

Implement methods:

- `registerUser(User user): boolean`
  - SQL: `INSERT INTO users(full_name,email,password_hash,phone,role) VALUES(?,?,?,?,?)`
  - Hash the password before insert.

- `loginUser(String email, String password): User`
  - SQL: `SELECT * FROM users WHERE email=?`
  - Verify password with hash check

- `getUserById(int userId): User`
- `updateUser(User user): boolean`
- `deleteUser(int userId): boolean`
- `getAllUsers(): List<User>`

### 7.2 CategoryDAO — required queries

- `addCategory(Category category): boolean`
- `getAllCategories(): List<Category>`
- `getCategoryById(int categoryId): Category`
- `updateCategory(Category category): boolean`
- `deleteCategory(int categoryId): boolean`

### 7.3 PostDAO — required queries

- `addPost(Post post): boolean`
- `updatePost(Post post): boolean`
- `deletePost(int postId): boolean`
- `getPostById(int postId): Post`
- `getAllPosts(): List<Post>`
- `getPostsByUser(int userId): List<Post>`
- `getPostsByCategory(int categoryId): List<Post>`
- `searchPosts(String keyword): List<Post>`
  - SQL like: `WHERE title LIKE ? OR description LIKE ?`

### 7.4 RequestDAO — required queries

- `sendRequest(Request request): boolean`
- `getRequestsByPost(int postId): List<Request>`
- `getRequestsBySender(int senderId): List<Request>`
- `updateRequestStatus(int requestId, String status): boolean`

### 7.5 MessageDAO (optional)

- `sendMessage(Message message): boolean`
- `getInbox(int userId): List<Message>`
- `getSentMessages(int userId): List<Message>`
- `getMessagesByPost(int postId): List<Message>`

---

## 8) Authentication module (Servlets + JSP)

### 8.1 Pages

Create in `webapp/`:
- `register.jsp`
- `login.jsp`

Both must have:
- `<form method="post" action="register">` or `action="login"`
- Fields:
  - Register: fullName, email, password, phone, role (student/admin) OR default student
  - Login: email, password

### 8.2 Servlets

Create in `src/controller/`:

- `RegisterServlet.java`
  - `doGet`: forward to `register.jsp`
  - `doPost`: read form fields, validate, call `UserDAO.registerUser`, then redirect to login.

- `LoginServlet.java`
  - `doGet`: forward to `login.jsp`
  - `doPost`: call `UserDAO.loginUser`
  - If success:
    - create session: `request.getSession(true)`
    - set: `session.setAttribute("user", user)`
    - redirect:
      - if admin → `admin-dashboard`
      - else → `home`

- `LogoutServlet.java`
  - invalidate session
  - redirect to login

### 8.3 Session rule (must)
On any servlet that requires login:
- if session is null OR `session.getAttribute("user") == null`
  - redirect to `login`

### 8.4 Role check rule
When accessing admin pages:
- get user from session
- if user role != `admin`
  - send 403 or redirect to home

---

## 9) Home + posts listing (ListPostsServlet)

### 9.1 Create page

- `home.jsp`
  - Shows navbar
  - Shows list of posts
  - Shows categories dropdown
  - Shows search bar

### 9.2 Servlet

- `ListPostsServlet.java`
  - `doGet`:
    - fetch categories (CategoryDAO)
    - fetch posts (PostDAO.getAllPosts)
    - set as request attributes
    - forward to `home.jsp`

---

## 10) Posts module (Create / Update / Delete / Details)

### 10.1 Pages

Create:
- `create-post.jsp`
- `edit-post.jsp`
- `post-details.jsp`

### 10.2 CreatePostServlet

Flow:
1. `doGet`: fetch categories list and forward to create-post page.
2. `doPost`:
   - read title, description, categoryId, postType, contactInfo
   - read current logged-in userId from session user
   - call `PostDAO.addPost`
   - redirect to `home`

### 10.3 UpdatePostServlet

Rules:
- Only the post owner OR admin can edit.

Flow:
1. `doGet`: load post by id, forward to edit page.
2. `doPost`: update and redirect to details.

### 10.4 DeletePostServlet

Rules:
- Only owner OR admin can delete.

Flow:
- delete by id, redirect home.

### 10.5 PostDetailsServlet

- loads one post
- loads requests for this post (optional)
- forward to details page

---

## 11) Categories module (filter)

### Page
- `category-results.jsp`

### Servlet
- `CategoryServlet.java`
  - reads `categoryId`
  - calls `PostDAO.getPostsByCategory(categoryId)`
  - forward results

---

## 12) Search & filter module

### Page
- `search-results.jsp`

### Servlet
- `SearchServlet.java`
  - reads:
    - `q` (keyword)
    - optional categoryId
    - optional postType
    - optional sort=latest
  - simplest version:
    - keyword search using `PostDAO.searchPosts(q)`
  - forward results

---

## 13) Requests module (respond to a post)

### Pages
- `request-form.jsp`
- `my-requests.jsp`
- `received-requests.jsp`

### Servlets

- `SendRequestServlet.java`
  - user selects a post and writes message
  - insert into requests with status pending

- `ViewRequestsServlet.java`
  - show **sent** requests by sender
  - show **received** requests by posts owned by the logged-in user (you can implement a DAO method for this)

- `UpdateRequestStatusServlet.java`
  - only post owner can accept/reject
  - update status accepted/rejected

---

## 14) Profile module

### Pages
- `profile.jsp`
- `edit-profile.jsp`
- `my-posts.jsp`

### Servlets
- `ProfileServlet.java` → shows user info
- `EditProfileServlet.java` → update name/email/phone
- `ChangePasswordServlet.java` → update password (hash)

---

## 15) Admin module (minimum required)

### Pages
- `admin-dashboard.jsp`
- `manage-users.jsp`
- `manage-posts.jsp`
- `manage-categories.jsp`

### Servlets
- `AdminDashboardServlet.java`
- `ManageUsersServlet.java`
- `ManagePostsServlet.java`
- `ManageCategoriesServlet.java`

Rules:
- All admin servlets must check role==admin.

Functions:
- list users, delete user
- list posts, delete post
- list categories, add/delete category

---

## 16) web.xml or annotations (choose one)

### Recommended: Annotations

Use `@WebServlet("/login")`, `@WebServlet("/register")`, etc.

Make sure JSP form action matches exactly.

Example mapping list:
- `/login`
- `/register`
- `/logout`
- `/home`
- `/posts/create`
- `/posts/edit`
- `/posts/delete`
- `/posts/details`
- `/categories`
- `/search`
- `/requests/send`
- `/requests/view`
- `/requests/update-status`
- `/admin/dashboard`

---

## 17) UI checklist (what every page must contain)

1. Navbar links:
   - Home
   - Create Post
   - My Profile
   - My Posts
   - Logout
   - (Admin) Dashboard
2. Show the logged-in user name on the navbar.
3. Use consistent CSS file: `webapp/css/style.css`

---

## 18) Testing checklist (do these in order)

1. Start MySQL.
2. Run SQL script → confirm tables exist.
3. Start Tomcat.
4. Open:
   - `http://localhost:8080/CampusConnect/register`
5. Register a student.
6. Login.
7. Create a category (if not seeded) or confirm categories exist.
8. Create a post.
9. Confirm it shows on home.
10. Search by keyword.
11. Filter by category.
12. Send a request to a post.
13. Accept/reject request as the post owner.
14. Create admin user (role admin) and verify admin dashboard pages.

---

## 19) Common mistakes (avoid)

- Using Tomcat 10 with `javax.servlet` imports (it will break). Use Tomcat 9 OR migrate to `jakarta.servlet`.
- Not closing JDBC resources → memory leaks.
- Storing raw passwords. Always store hash.
- Not checking session in protected servlets.
- Not checking ownership/role before edit/delete.

---

## 20) Deliverables (what to submit)

- Source code (src)
- webapp JSP pages + CSS
- `database/campusconnect.sql`
- Updated README (this file)

---

## 21) Next improvements (optional)

- Profile image upload
- Pagination for posts
- Full messaging inbox
- Upgrade to Spring Boot (future)