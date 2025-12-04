---
layout: default
title: Frontend Implementation
permalink: /flows/frontend-implementation/
---

# Frontend Implementation

[← Quay lại tổng quan](../account-verification-activation/)

---

## 📋 Nội dung

Phần này giải thích frontend implementation của Account Verification Flow:
- React component structure
- Form handling với React Hook Form
- File upload UI
- Admin dashboard
- API integration với Axios
- State management
- Error handling

---

## 1. Technology Stack

### Core Libraries

```json
{
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "react-router-dom": "^6.14.0",
    "typescript": "^5.0.2",
    "axios": "^1.4.0",
    "react-hook-form": "^7.45.0",
    "react-hot-toast": "^2.4.1",
    "tailwindcss": "^3.3.0"
  }
}
```

### Project Structure

```
frontend/src/
├── components/
│   ├── auth/
│   │   ├── Login.tsx
│   │   └── Register.tsx
│   ├── verification/
│   │   ├── RiderVehicleVerification.tsx
│   │   ├── DriverVehicleVerification.tsx
│   │   └── DocumentUpload.tsx
│   └── admin/
│       └── VerificationManagement.tsx
├── services/
│   ├── authService.ts
│   ├── verificationService.ts
│   └── api.ts
├── contexts/
│   └── AuthContext.tsx
├── types/
│   └── index.ts
└── utils/
    └── validation.ts
```

---

## 2. Authentication Flow (Login Component)

### Login.tsx

```typescript
import React, { useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { useAuth } from '@/contexts/AuthContext';
import { toast } from 'react-hot-toast';

interface LoginForm {
  email: string;
  password: string;
}

export const Login: React.FC = () => {
  const [formData, setFormData] = useState<LoginForm>({
    email: '',
    password: ''
  });
  const [loading, setLoading] = useState(false);
  const [showPassword, setShowPassword] = useState(false);
  
  const navigate = useNavigate();
  const { login } = useAuth();

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setLoading(true);

    try {
      const response = await login(formData.email, formData.password);
      
      // Store token in localStorage
      localStorage.setItem('access_token', response.access_token);
      localStorage.setItem('user', JSON.stringify(response.user));
      
      toast.success('Đăng nhập thành công!');
      
      // Redirect based on role
      if (response.user.roles.includes('ROLE_ADMIN')) {
        navigate('/admin/verifications');
      } else {
        navigate('/dashboard');
      }
      
    } catch (error: any) {
      const message = error.response?.data?.message || 'Đăng nhập thất bại';
      
      // Handle specific error cases
      if (error.response?.status === 401) {
        toast.error('Email hoặc mật khẩu không đúng');
      } else if (error.response?.status === 403) {
        toast.error('Tài khoản chưa được kích hoạt. Vui lòng chờ admin duyệt.');
      } else if (error.response?.status === 429) {
        toast.error('Bạn đã nhập sai quá nhiều lần. Vui lòng thử lại sau 30 phút.');
      } else {
        toast.error(message);
      }
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="min-h-screen flex items-center justify-center bg-gray-50">
      <div className="max-w-md w-full bg-white rounded-lg shadow-md p-8">
        <h2 className="text-2xl font-bold text-center mb-6">
          Đăng nhập MSSUS
        </h2>
        
        <form onSubmit={handleSubmit} className="space-y-4">
          {/* Email input */}
          <div>
            <label className="block text-sm font-medium text-gray-700 mb-1">
              Email
            </label>
            <input
              type="email"
              required
              value={formData.email}
              onChange={(e) => setFormData({ ...formData, email: e.target.value })}
              className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
              placeholder="student@fpt.edu.vn"
            />
          </div>

          {/* Password input */}
          <div>
            <label className="block text-sm font-medium text-gray-700 mb-1">
              Mật khẩu
            </label>
            <div className="relative">
              <input
                type={showPassword ? 'text' : 'password'}
                required
                value={formData.password}
                onChange={(e) => setFormData({ ...formData, password: e.target.value })}
                className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
                placeholder="********"
              />
              <button
                type="button"
                onClick={() => setShowPassword(!showPassword)}
                className="absolute right-3 top-1/2 -translate-y-1/2 text-gray-500"
              >
                {showPassword ? '🙈' : '👁️'}
              </button>
            </div>
          </div>

          {/* Submit button */}
          <button
            type="submit"
            disabled={loading}
            className={`w-full py-2 px-4 rounded-md text-white font-medium ${
              loading 
                ? 'bg-gray-400 cursor-not-allowed' 
                : 'bg-blue-600 hover:bg-blue-700'
            }`}
          >
            {loading ? 'Đang xử lý...' : 'Đăng nhập'}
          </button>
        </form>

        <p className="mt-4 text-center text-sm text-gray-600">
          Chưa có tài khoản?{' '}
          <a href="/register" className="text-blue-600 hover:underline">
            Đăng ký ngay
          </a>
        </p>
      </div>
    </div>
  );
};
```

---

## 3. Document Upload Component

### RiderVehicleVerification.tsx

```typescript
import React, { useState } from 'react';
import { useForm } from 'react-hook-form';
import { toast } from 'react-hot-toast';
import { verificationService } from '@/services/verificationService';

interface StudentVerificationForm {
  studentIdFront: FileList;
  studentIdBack: FileList;
  studentCardUrl: FileList;
}

export const RiderVehicleVerification: React.FC = () => {
  const { register, handleSubmit, formState: { errors }, watch } = 
    useForm<StudentVerificationForm>();
  
  const [uploading, setUploading] = useState(false);
  const [previewUrls, setPreviewUrls] = useState<{
    studentIdFront?: string;
    studentIdBack?: string;
    studentCardUrl?: string;
  }>({});

  // Watch file inputs for preview
  const studentIdFront = watch('studentIdFront');
  const studentIdBack = watch('studentIdBack');
  const studentCardUrl = watch('studentCardUrl');

  React.useEffect(() => {
    if (studentIdFront?.[0]) {
      setPreviewUrls(prev => ({
        ...prev,
        studentIdFront: URL.createObjectURL(studentIdFront[0])
      }));
    }
  }, [studentIdFront]);

  React.useEffect(() => {
    if (studentIdBack?.[0]) {
      setPreviewUrls(prev => ({
        ...prev,
        studentIdBack: URL.createObjectURL(studentIdBack[0])
      }));
    }
  }, [studentIdBack]);

  React.useEffect(() => {
    if (studentCardUrl?.[0]) {
      setPreviewUrls(prev => ({
        ...prev,
        studentCardUrl: URL.createObjectURL(studentCardUrl[0])
      }));
    }
  }, [studentCardUrl]);

  const validateFile = (file: File): string | true => {
    // Check file size (max 10MB)
    const maxSize = 10 * 1024 * 1024;
    if (file.size > maxSize) {
      return 'Kích thước file tối đa 10MB';
    }

    // Check file type
    const allowedTypes = ['image/jpeg', 'image/png', 'application/pdf'];
    if (!allowedTypes.includes(file.type)) {
      return 'Chỉ chấp nhận file JPG, PNG, hoặc PDF';
    }

    return true;
  };

  const onSubmit = async (data: StudentVerificationForm) => {
    setUploading(true);

    try {
      // Validate files
      const frontValidation = validateFile(data.studentIdFront[0]);
      const backValidation = validateFile(data.studentIdBack[0]);
      const cardValidation = validateFile(data.studentCardUrl[0]);

      if (frontValidation !== true) {
        toast.error(`Mặt trước CCCD: ${frontValidation}`);
        return;
      }
      if (backValidation !== true) {
        toast.error(`Mặt sau CCCD: ${backValidation}`);
        return;
      }
      if (cardValidation !== true) {
        toast.error(`Thẻ sinh viên: ${cardValidation}`);
        return;
      }

      // Create FormData
      const formData = new FormData();
      formData.append('studentIdFront', data.studentIdFront[0]);
      formData.append('studentIdBack', data.studentIdBack[0]);
      formData.append('studentCardUrl', data.studentCardUrl[0]);

      // Submit to API
      const response = await verificationService.submitStudentVerification(formData);

      toast.success('Nộp hồ sơ thành công! Vui lòng chờ admin duyệt.');
      
      // Redirect to dashboard
      window.location.href = '/dashboard';

    } catch (error: any) {
      const message = error.response?.data?.message || 'Có lỗi xảy ra';
      toast.error(message);
    } finally {
      setUploading(false);
    }
  };

  return (
    <div className="max-w-4xl mx-auto p-6">
      <h1 className="text-2xl font-bold mb-6">Xác minh tài khoản Rider</h1>
      
      <div className="bg-yellow-50 border border-yellow-200 rounded-md p-4 mb-6">
        <p className="text-sm text-yellow-800">
          ⚠️ <strong>Lưu ý:</strong> Vui lòng chụp ảnh rõ ràng, đủ sáng, không bị mờ hoặc che khuất. 
          Ảnh CCCD phải hiển thị đầy đủ thông tin.
        </p>
      </div>

      <form onSubmit={handleSubmit(onSubmit)} className="space-y-6">
        {/* Student ID Front */}
        <div>
          <label className="block text-sm font-medium text-gray-700 mb-2">
            CCCD/CMND (Mặt trước) <span className="text-red-500">*</span>
          </label>
          <input
            type="file"
            accept="image/jpeg,image/png,application/pdf"
            {...register('studentIdFront', { required: 'Bắt buộc tải lên' })}
            className="block w-full text-sm text-gray-500 file:mr-4 file:py-2 file:px-4 file:rounded-md file:border-0 file:text-sm file:font-semibold file:bg-blue-50 file:text-blue-700 hover:file:bg-blue-100"
          />
          {errors.studentIdFront && (
            <p className="mt-1 text-sm text-red-600">{errors.studentIdFront.message}</p>
          )}
          {previewUrls.studentIdFront && (
            <div className="mt-2">
              <img 
                src={previewUrls.studentIdFront} 
                alt="Preview" 
                className="w-64 h-40 object-cover rounded border"
              />
            </div>
          )}
        </div>

        {/* Student ID Back */}
        <div>
          <label className="block text-sm font-medium text-gray-700 mb-2">
            CCCD/CMND (Mặt sau) <span className="text-red-500">*</span>
          </label>
          <input
            type="file"
            accept="image/jpeg,image/png,application/pdf"
            {...register('studentIdBack', { required: 'Bắt buộc tải lên' })}
            className="block w-full text-sm text-gray-500 file:mr-4 file:py-2 file:px-4 file:rounded-md file:border-0 file:text-sm file:font-semibold file:bg-blue-50 file:text-blue-700 hover:file:bg-blue-100"
          />
          {errors.studentIdBack && (
            <p className="mt-1 text-sm text-red-600">{errors.studentIdBack.message}</p>
          )}
          {previewUrls.studentIdBack && (
            <div className="mt-2">
              <img 
                src={previewUrls.studentIdBack} 
                alt="Preview" 
                className="w-64 h-40 object-cover rounded border"
              />
            </div>
          )}
        </div>

        {/* Student Card */}
        <div>
          <label className="block text-sm font-medium text-gray-700 mb-2">
            Thẻ sinh viên <span className="text-red-500">*</span>
          </label>
          <input
            type="file"
            accept="image/jpeg,image/png,application/pdf"
            {...register('studentCardUrl', { required: 'Bắt buộc tải lên' })}
            className="block w-full text-sm text-gray-500 file:mr-4 file:py-2 file:px-4 file:rounded-md file:border-0 file:text-sm file:font-semibold file:bg-blue-50 file:text-blue-700 hover:file:bg-blue-100"
          />
          {errors.studentCardUrl && (
            <p className="mt-1 text-sm text-red-600">{errors.studentCardUrl.message}</p>
          )}
          {previewUrls.studentCardUrl && (
            <div className="mt-2">
              <img 
                src={previewUrls.studentCardUrl} 
                alt="Preview" 
                className="w-64 h-40 object-cover rounded border"
              />
            </div>
          )}
        </div>

        {/* Submit button */}
        <button
          type="submit"
          disabled={uploading}
          className={`w-full py-3 px-4 rounded-md text-white font-medium ${
            uploading 
              ? 'bg-gray-400 cursor-not-allowed' 
              : 'bg-blue-600 hover:bg-blue-700'
          }`}
        >
          {uploading ? 'Đang tải lên...' : 'Nộp hồ sơ'}
        </button>
      </form>
    </div>
  );
};
```

---

## 4. Admin Verification Dashboard

### VerificationManagement.tsx

```typescript
import React, { useEffect, useState } from 'react';
import { toast } from 'react-hot-toast';
import { verificationService } from '@/services/verificationService';

interface Verification {
  id: number;
  user: {
    fullName: string;
    email: string;
  };
  verificationType: 'STUDENT_ID' | 'DRIVER_LICENSE' | 'VEHICLE_REGISTRATION';
  status: 'PENDING' | 'APPROVED' | 'REJECTED';
  studentIdFront?: string;
  studentIdBack?: string;
  studentCardUrl?: string;
  driverLicenseUrl?: string;
  vehicleRegistrationUrl?: string;
  createdAt: string;
}

export const VerificationManagement: React.FC = () => {
  const [verifications, setVerifications] = useState<Verification[]>([]);
  const [loading, setLoading] = useState(true);
  const [selectedVerification, setSelectedVerification] = useState<Verification | null>(null);
  const [showModal, setShowModal] = useState(false);
  const [rejectReason, setRejectReason] = useState('');
  const [actionLoading, setActionLoading] = useState(false);

  useEffect(() => {
    fetchVerifications();
  }, []);

  const fetchVerifications = async () => {
    setLoading(true);
    try {
      const response = await verificationService.getVerifications('PENDING');
      setVerifications(response.data);
    } catch (error) {
      toast.error('Không thể tải danh sách xác minh');
    } finally {
      setLoading(false);
    }
  };

  const handleApprove = async (verificationId: number) => {
    if (!window.confirm('Bạn chắc chắn muốn duyệt hồ sơ này?')) {
      return;
    }

    setActionLoading(true);
    try {
      await verificationService.approveVerification(verificationId);
      toast.success('Đã duyệt hồ sơ thành công!');
      
      // Remove from list
      setVerifications(prev => prev.filter(v => v.id !== verificationId));
    } catch (error: any) {
      const message = error.response?.data?.message || 'Có lỗi xảy ra';
      toast.error(message);
    } finally {
      setActionLoading(false);
    }
  };

  const handleReject = async () => {
    if (!selectedVerification) return;
    
    if (!rejectReason.trim()) {
      toast.error('Vui lòng nhập lý do từ chối');
      return;
    }

    setActionLoading(true);
    try {
      await verificationService.rejectVerification(
        selectedVerification.id,
        rejectReason
      );
      toast.success('Đã từ chối hồ sơ');
      
      // Remove from list
      setVerifications(prev => prev.filter(v => v.id !== selectedVerification.id));
      
      // Close modal
      setShowModal(false);
      setRejectReason('');
      setSelectedVerification(null);
    } catch (error: any) {
      const message = error.response?.data?.message || 'Có lỗi xảy ra';
      toast.error(message);
    } finally {
      setActionLoading(false);
    }
  };

  const openRejectModal = (verification: Verification) => {
    setSelectedVerification(verification);
    setShowModal(true);
  };

  if (loading) {
    return (
      <div className="flex items-center justify-center h-screen">
        <p className="text-gray-500">Đang tải...</p>
      </div>
    );
  }

  return (
    <div className="max-w-7xl mx-auto p-6">
      <h1 className="text-2xl font-bold mb-6">Quản lý xác minh tài khoản</h1>

      {verifications.length === 0 ? (
        <div className="bg-gray-50 rounded-lg p-8 text-center">
          <p className="text-gray-500">Không có hồ sơ chờ duyệt</p>
        </div>
      ) : (
        <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
          {verifications.map((verification) => (
            <div key={verification.id} className="bg-white rounded-lg shadow-md p-6">
              {/* User info */}
              <div className="mb-4">
                <h3 className="font-semibold text-lg">{verification.user.fullName}</h3>
                <p className="text-sm text-gray-500">{verification.user.email}</p>
                <span className="inline-block mt-2 px-2 py-1 text-xs rounded bg-yellow-100 text-yellow-800">
                  {verification.verificationType.replace('_', ' ')}
                </span>
              </div>

              {/* Documents preview */}
              <div className="space-y-2 mb-4">
                {verification.studentIdFront && (
                  <div>
                    <p className="text-xs text-gray-600 mb-1">CCCD (Mặt trước)</p>
                    <img 
                      src={verification.studentIdFront} 
                      alt="Student ID Front"
                      className="w-full h-32 object-cover rounded cursor-pointer hover:opacity-80"
                      onClick={() => window.open(verification.studentIdFront, '_blank')}
                    />
                  </div>
                )}
                {verification.studentIdBack && (
                  <div>
                    <p className="text-xs text-gray-600 mb-1">CCCD (Mặt sau)</p>
                    <img 
                      src={verification.studentIdBack} 
                      alt="Student ID Back"
                      className="w-full h-32 object-cover rounded cursor-pointer hover:opacity-80"
                      onClick={() => window.open(verification.studentIdBack, '_blank')}
                    />
                  </div>
                )}
                {verification.studentCardUrl && (
                  <div>
                    <p className="text-xs text-gray-600 mb-1">Thẻ sinh viên</p>
                    <img 
                      src={verification.studentCardUrl} 
                      alt="Student Card"
                      className="w-full h-32 object-cover rounded cursor-pointer hover:opacity-80"
                      onClick={() => window.open(verification.studentCardUrl, '_blank')}
                    />
                  </div>
                )}
              </div>

              {/* Actions */}
              <div className="flex gap-2">
                <button
                  onClick={() => handleApprove(verification.id)}
                  disabled={actionLoading}
                  className="flex-1 py-2 px-4 bg-green-600 text-white rounded-md hover:bg-green-700 disabled:bg-gray-400"
                >
                  Duyệt
                </button>
                <button
                  onClick={() => openRejectModal(verification)}
                  disabled={actionLoading}
                  className="flex-1 py-2 px-4 bg-red-600 text-white rounded-md hover:bg-red-700 disabled:bg-gray-400"
                >
                  Từ chối
                </button>
              </div>

              {/* Timestamp */}
              <p className="text-xs text-gray-400 mt-3">
                Nộp lúc: {new Date(verification.createdAt).toLocaleString('vi-VN')}
              </p>
            </div>
          ))}
        </div>
      )}

      {/* Reject Modal */}
      {showModal && (
        <div className="fixed inset-0 bg-black bg-opacity-50 flex items-center justify-center z-50">
          <div className="bg-white rounded-lg p-6 max-w-md w-full">
            <h2 className="text-xl font-bold mb-4">Từ chối hồ sơ</h2>
            
            <p className="text-sm text-gray-600 mb-4">
              Người dùng: <strong>{selectedVerification?.user.fullName}</strong>
            </p>

            <label className="block text-sm font-medium text-gray-700 mb-2">
              Lý do từ chối <span className="text-red-500">*</span>
            </label>
            <textarea
              value={rejectReason}
              onChange={(e) => setRejectReason(e.target.value)}
              className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-red-500"
              rows={4}
              placeholder="Ví dụ: Ảnh CCCD bị mờ, vui lòng chụp lại"
            />

            <div className="flex gap-2 mt-6">
              <button
                onClick={() => {
                  setShowModal(false);
                  setRejectReason('');
                  setSelectedVerification(null);
                }}
                disabled={actionLoading}
                className="flex-1 py-2 px-4 bg-gray-200 text-gray-800 rounded-md hover:bg-gray-300 disabled:bg-gray-100"
              >
                Hủy
              </button>
              <button
                onClick={handleReject}
                disabled={actionLoading || !rejectReason.trim()}
                className="flex-1 py-2 px-4 bg-red-600 text-white rounded-md hover:bg-red-700 disabled:bg-gray-400"
              >
                {actionLoading ? 'Đang xử lý...' : 'Xác nhận từ chối'}
              </button>
            </div>
          </div>
        </div>
      )}
    </div>
  );
};
```

---

## 5. API Service Layer

### api.ts - Axios Configuration

```typescript
import axios, { AxiosError, AxiosResponse } from 'axios';
import { toast } from 'react-hot-toast';

const api = axios.create({
  baseURL: process.env.REACT_APP_API_BASE_URL || 'http://localhost:8080/api/v1',
  timeout: 30000, // 30 seconds
  headers: {
    'Content-Type': 'application/json',
  },
});

// Request interceptor - Add JWT token
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('access_token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => {
    return Promise.reject(error);
  }
);

// Response interceptor - Handle errors globally
api.interceptors.response.use(
  (response: AxiosResponse) => {
    return response;
  },
  (error: AxiosError) => {
    if (error.response) {
      const status = error.response.status;
      
      switch (status) {
        case 401:
          // Unauthorized - clear token and redirect to login
          localStorage.removeItem('access_token');
          localStorage.removeItem('user');
          window.location.href = '/login';
          toast.error('Phiên đăng nhập hết hạn. Vui lòng đăng nhập lại.');
          break;
          
        case 403:
          toast.error('Bạn không có quyền truy cập');
          break;
          
        case 404:
          toast.error('Không tìm thấy dữ liệu');
          break;
          
        case 500:
          toast.error('Lỗi máy chủ. Vui lòng thử lại sau.');
          break;
          
        default:
          // Let component handle specific errors
          break;
      }
    } else if (error.request) {
      toast.error('Không thể kết nối tới server');
    }
    
    return Promise.reject(error);
  }
);

export default api;
```

### verificationService.ts

```typescript
import api from './api';

export interface VerificationResponse {
  id: number;
  userId: number;
  verificationType: string;
  status: string;
  createdAt: string;
}

export const verificationService = {
  // Submit student verification
  submitStudentVerification: async (formData: FormData) => {
    const response = await api.post<VerificationResponse>(
      '/me/student-verifications',
      formData,
      {
        headers: {
          'Content-Type': 'multipart/form-data',
        },
      }
    );
    return response.data;
  },

  // Submit driver license
  submitDriverLicense: async (formData: FormData) => {
    const response = await api.post<VerificationResponse>(
      '/me/driver-verifications/driver-license',
      formData,
      {
        headers: {
          'Content-Type': 'multipart/form-data',
        },
      }
    );
    return response.data;
  },

  // Submit vehicle registration
  submitVehicleRegistration: async (formData: FormData) => {
    const response = await api.post<VerificationResponse>(
      '/me/driver-verifications/vehicle-registration',
      formData,
      {
        headers: {
          'Content-Type': 'multipart/form-data',
        },
      }
    );
    return response.data;
  },

  // Get all verifications (admin)
  getVerifications: async (status?: string) => {
    const response = await api.get('/verification', {
      params: { status },
    });
    return response.data;
  },

  // Approve verification (admin)
  approveVerification: async (verificationId: number) => {
    const response = await api.post('/verification/approve', {
      verificationId,
    });
    return response.data;
  },

  // Reject verification (admin)
  rejectVerification: async (verificationId: number, reason: string) => {
    const response = await api.post('/verification/reject', {
      verificationId,
      reason,
    });
    return response.data;
  },
};
```

---

## 6. State Management - Auth Context

### AuthContext.tsx

```typescript
import React, { createContext, useContext, useState, useEffect } from 'react';
import { authService } from '@/services/authService';

interface User {
  id: number;
  email: string;
  fullName: string;
  roles: string[];
  status: string;
}

interface AuthContextType {
  user: User | null;
  loading: boolean;
  login: (email: string, password: string) => Promise<any>;
  logout: () => void;
  isAuthenticated: boolean;
}

const AuthContext = createContext<AuthContextType | undefined>(undefined);

export const AuthProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // Check if user is already logged in
    const storedUser = localStorage.getItem('user');
    const token = localStorage.getItem('access_token');
    
    if (storedUser && token) {
      setUser(JSON.parse(storedUser));
    }
    
    setLoading(false);
  }, []);

  const login = async (email: string, password: string) => {
    const response = await authService.login(email, password);
    setUser(response.user);
    return response;
  };

  const logout = () => {
    localStorage.removeItem('access_token');
    localStorage.removeItem('user');
    setUser(null);
    window.location.href = '/login';
  };

  return (
    <AuthContext.Provider
      value={{
        user,
        loading,
        login,
        logout,
        isAuthenticated: !!user,
      }}
    >
      {children}
    </AuthContext.Provider>
  );
};

export const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
};
```

---

## 7. Error Handling & Toasts

### Toast Notifications

```typescript
import { toast } from 'react-hot-toast';

// Success toast
toast.success('Đăng nhập thành công!');

// Error toast
toast.error('Email hoặc mật khẩu không đúng');

// Loading toast
const loadingToast = toast.loading('Đang xử lý...');
// Later: toast.dismiss(loadingToast);

// Custom toast
toast.custom((t) => (
  <div className={`bg-white rounded-lg shadow-lg p-4 ${t.visible ? 'animate-enter' : 'animate-leave'}`}>
    <p>Hồ sơ của bạn đang được xét duyệt...</p>
  </div>
));
```

---

## 8. Type Definitions

### types/index.ts

```typescript
export enum UserStatus {
  EMAIL_VERIFYING = 'EMAIL_VERIFYING',
  PENDING = 'PENDING',
  ACTIVE = 'ACTIVE',
  INACTIVE = 'INACTIVE',
  BANNED = 'BANNED',
}

export enum VerificationStatus {
  PENDING = 'PENDING',
  APPROVED = 'APPROVED',
  REJECTED = 'REJECTED',
}

export enum VerificationType {
  STUDENT_ID = 'STUDENT_ID',
  DRIVER_LICENSE = 'DRIVER_LICENSE',
  VEHICLE_REGISTRATION = 'VEHICLE_REGISTRATION',
}

export interface User {
  id: number;
  email: string;
  phone: string;
  fullName: string;
  dateOfBirth: string;
  status: UserStatus;
  roles: string[];
  createdAt: string;
  updatedAt: string;
}

export interface Verification {
  id: number;
  userId: number;
  verificationType: VerificationType;
  status: VerificationStatus;
  studentIdFront?: string;
  studentIdBack?: string;
  studentCardUrl?: string;
  driverLicenseUrl?: string;
  vehicleRegistrationUrl?: string;
  rejectionReason?: string;
  reviewedAt?: string;
  createdAt: string;
}
```

---

[← Quay lại tổng quan](../account-verification-activation/) | [Đọc tiếp: Security & Validation →](../security-validation/)
