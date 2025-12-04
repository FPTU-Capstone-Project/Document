---
layout: default
title: Trang chủ
---

# Hệ thống chia sẻ xe máy cho sinh viên (MSSUS)

## Giới thiệu dự án

Đây là tài liệu kỹ thuật chi tiết của dự án Capstone **Motorbike Sharing System for University Students (MSSUS)** được phát triển tại Đại học FPT. Hệ thống cung cấp giải pháp chia sẻ xe máy an toàn, hiệu quả dành riêng cho cộng đồng sinh viên.

Documentation này được tạo ra với mục đích:
- **Giải thích chi tiết** từng nghiệp vụ trong hệ thống
- **Phân tích kỹ thuật** cách triển khai Backend và Frontend
- **Trả lời câu hỏi** Why, What, How cho mỗi quyết định thiết kế
- **Chuẩn bị bảo vệ** đồ án với câu hỏi phản biện và cách trả lời

---

## Tech Stack

### Backend
- **Framework**: Java Spring Boot 3.x
- **Database**: PostgreSQL
- **Message Queue**: RabbitMQ
- **Cache**: Redis
- **Security**: Spring Security + JWT
- **File Storage**: AWS S3 / Local Storage
- **API Docs**: Swagger/OpenAPI

### Frontend
- **Framework**: React 18 + TypeScript
- **UI Library**: Tailwind CSS
- **State Management**: React Context + Custom Hooks
- **HTTP Client**: Axios
- **Form Validation**: React Hook Form + Yup

### Infrastructure
- **Containers**: Docker + Docker Compose
- **Cloud Providers**: AWS, Google Cloud Platform
- **IaC**: Terraform
- **CI/CD**: GitHub Actions
- **Monitoring**: Prometheus + Grafana (planned)

---

## Main System Flows

### [1. Account Verification & Activation Flow](flows/account-verification-activation/)
**Status**: ✅ Đã hoàn thành

Chi tiết về quy trình:
- Đăng ký tài khoản người dùng
- Xác thực email/OTP
- Upload giấy tờ định danh (Thẻ sinh viên/CMND)
- Admin xem và duyệt/từ chối hồ sơ
- Kích hoạt tài khoản và gửi email thông báo
- Người dùng đăng nhập và sử dụng dịch vụ

**[➜ Xem chi tiết flow](flows/account-verification-activation/)**

---

### 2. Ride Booking & Matching Flow
**Status**: ⏳ Sắp ra mắt

Quy trình đặt chuyến xe, thuật toán ghép đôi người đi/người chở, tính toán cước phí, và xử lý yêu cầu.

---

### 3. Payment & Wallet System
**Status**: ⏳ Sắp ra mắt

Hệ thống ví điện tử, nạp tiền, thanh toán chuyến xe, hoàn tiền, và quản lý giao dịch.

---

### 4. SOS & Emergency Management
**Status**: ⏳ Sắp ra mắt

Cảnh báo khẩn cấp, xử lý sự cố an toàn, thông báo cho admin và liên hệ cơ quan chức năng.

---

## Đặc điểm của Documentation

### 📚 Phân tích nghiệp vụ chi tiết
Giải thích từng bước trong quy trình, lý do thiết kế, quyết định kiến trúc, và trade-offs.

### 🔧 Technical Implementation
Code examples, API endpoints, database schema, luồng xử lý dữ liệu, và error handling.

### 💡 Why, What, How
Trả lời 3 câu hỏi cốt lõi:
- **Why** (Tại sao): Lý do quyết định thiết kế
- **What** (Cái gì): Chức năng và yêu cầu
- **How** (Làm thế nào): Cách triển khai kỹ thuật

### ❓ Câu hỏi phản biện
Tập hợp câu hỏi thường gặp khi bảo vệ đồ án và cách trả lời chuyên nghiệp.

### 🎯 Frontend & Backend
Phân tích cả hai phía: UI/UX interaction và business logic xử lý phía server.

### 🔒 Security & Validation
Cơ chế bảo mật, xác thực JWT, phân quyền RBAC, validation, và xử lý lỗi.

---

## Cách sử dụng Documentation

1. **Đọc từng Flow**: Bắt đầu từ [Account Verification & Activation](flows/account-verification-activation/)
2. **Hiểu nghiệp vụ trước**: Đọc phần Business Logic trước khi đi vào kỹ thuật
3. **Xem code examples**: Tham khảo các đoạn code thực tế trong dự án
4. **Chuẩn bị câu hỏi**: Đọc phần Defense Questions và luyện tập trả lời
5. **Practice**: Thử giải thích flow cho người khác để kiểm tra hiểu biết

---

## Liên hệ & Đóng góp

- **GitHub Repository**: [FPTU-Capstone-Project](https://github.com/FPTU-Capstone-Project)
- **Project Team**: FPT University Capstone 2025
- **Documentation**: Được tạo tự động và cập nhật thường xuyên

---

**Lưu ý**: Documentation này được thiết kế để giúp bất kỳ ai không có insight về dự án có thể hiểu, thuyết trình, và bảo vệ thành công trước hội đồng.
