# MSSUS - Motorbike Sharing System Documentation

[![GitHub Pages](https://img.shields.io/badge/GitHub%20Pages-Live-brightgreen)](https://fptu-capstone-project.github.io/Document/)
[![Jekyll](https://img.shields.io/badge/Built%20with-Jekyll-red)](https://jekyllrb.com/)
[![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

## 📚 About

Tài liệu kỹ thuật toàn diện về hệ thống **Motorbike Sharing System (MSSUS)** - đồ án tốt nghiệp tại FPT University. Documentation này được thiết kế để giúp bạn hiểu sâu về nghiệp vụ và kỹ thuật triển khai, đủ chi tiết để bảo vệ đồ án một cách tự tin.

## 🌐 Live Site

**Truy cập tài liệu tại:** https://fptu-capstone-project.github.io/Document/

## 📖 Nội dung chính

### 1. Account Verification & Activation Flow

Chi tiết đầy đủ về quy trình xác minh và kích hoạt tài khoản:

- **[Tổng quan Flow](https://fptu-capstone-project.github.io/Document/flows/account-verification-activation/)** - Giới thiệu 5 bước của flow
- **[Business Logic](https://fptu-capstone-project.github.io/Document/flows/account-verification-activation/business-logic/)** - Nghiệp vụ chi tiết từng bước
- **[Backend Implementation](https://fptu-capstone-project.github.io/Document/flows/account-verification-activation/backend-implementation/)** - Database, API, Service Layer
- **[Backend APIs](https://fptu-capstone-project.github.io/Document/flows/account-verification-activation/backend-implementation-api/)** - Admin endpoints, Login, Email service
- **[Frontend Implementation](https://fptu-capstone-project.github.io/Document/flows/account-verification-activation/frontend-implementation/)** - React components, Forms, File upload, Admin dashboard
- **[Security & Validation](https://fptu-capstone-project.github.io/Document/flows/account-verification-activation/security-validation/)** - JWT, File validation, XSS/SQL injection prevention
- **[Câu hỏi phản biện](https://fptu-capstone-project.github.io/Document/flows/account-verification-activation/defense-questions/)** - 20+ câu hỏi thường gặp khi bảo vệ

### Coming Soon

- Driver Matching & Routing Flow
- Wallet & Payment Flow
- Rating & Review System Flow

## 🛠️ Tech Stack

### Backend
- **Framework:** Spring Boot 3.2
- **Database:** PostgreSQL 15
- **Authentication:** JWT (JSON Web Token)
- **File Storage:** AWS S3
- **Email:** Spring Mail + Thymeleaf
- **Migration:** Flyway

### Frontend
- **Framework:** React 18 + TypeScript
- **Styling:** Tailwind CSS
- **Forms:** React Hook Form
- **HTTP Client:** Axios
- **Notifications:** React Hot Toast

### Infrastructure
- **Containerization:** Docker
- **Cloud:** AWS EC2, GCP Compute Engine
- **IaC:** Terraform
- **CI/CD:** GitHub Actions
- **Documentation:** Jekyll + GitHub Pages

## 🚀 Local Development (Documentation Site)

### Prerequisites

- Ruby 2.7+
- Bundler

### Setup

```bash
# Clone repository
git clone https://github.com/FPTU-Capstone-Project/Document.git
cd Document

# Install dependencies
bundle install

# Run local server
bundle exec jekyll serve

# Open browser
open http://localhost:4000
```

### Build

```bash
# Build static site
bundle exec jekyll build

# Output in _site/ directory
```

## 📂 Repository Structure

```
Document/
├── _config.yml              # Jekyll configuration
├── index.md                 # Homepage
├── _flows/                  # Flow documentation
│   ├── account-verification-activation.md
│   ├── business-logic.md
│   ├── backend-implementation.md
│   ├── backend-implementation-api.md
│   ├── frontend-implementation.md
│   ├── security-validation.md
│   └── defense-questions.md
├── assets/                  # CSS, JS, images
│   └── css/
│       └── style.css
└── _layouts/                # Jekyll layouts (optional)
    └── default.html
```

## 🎯 Features

- ✅ **Comprehensive:** Covers business logic, technical implementation, security
- ✅ **Detailed:** Code examples from actual project (not pseudocode)
- ✅ **Modular:** Split into multiple linked files for easy navigation
- ✅ **Searchable:** Full-text search (coming soon)
- ✅ **Responsive:** Mobile-friendly design
- ✅ **Defense-ready:** Includes anticipated questions and answers

## 👥 Team

- **Project:** MSSUS - Motorbike Sharing System
- **University:** FPT University
- **Semester:** Fall 2024

## 📝 License

This documentation is licensed under the MIT License.

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

### How to contribute:

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

## 📞 Contact

For questions or feedback, please open an issue in this repository.

---

**⭐ Star this repo if you find it helpful!**
