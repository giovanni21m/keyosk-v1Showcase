# Code Snippets & Technical Examples

## 🎯 Purpose

This document contains carefully curated code snippets that demonstrate technical skills and architectural patterns without revealing any business logic or proprietary information.

## 📋 Table of Contents

1. [TypeScript & React Patterns](#typescript--react-patterns)
2. [Security Implementation](#security-implementation)
3. [Database & API Design](#database--api-design)
4. [Testing Strategies](#testing-strategies)
5. [Performance Optimization](#performance-optimization)

---

## TypeScript & React Patterns

### 1. Advanced Form Component with Validation

```typescript
// components/forms/UserForm.tsx
"use client";
import { useForm } from "react-hook-form";
import { z } from "zod";
import { zodResolver } from "@hookform/resolvers/zod";

// Type-safe schema definition
const UserSchema = z.object({
  name: z.string().min(2, "Name must be at least 2 characters"),
  email: z.string().email("Invalid email format"),
  role: z.enum(["admin", "user", "moderator"]),
  preferences: z.object({
    notifications: z.boolean().default(true),
    theme: z.enum(["light", "dark"]).default("light")
  }).optional()
});

type UserFormValues = z.infer<typeof UserSchema>;

interface UserFormProps {
  onSubmit: (data: UserFormValues) => Promise<void>;
  initialData?: Partial<UserFormValues>;
  loading?: boolean;
}

export default function UserForm({ onSubmit, initialData, loading }: UserFormProps) {
  const { register, handleSubmit, formState, watch, setError } = useForm<UserFormValues>({
    resolver: zodResolver(UserSchema),
    defaultValues: {
      preferences: {
        notifications: true,
        theme: "light"
      },
      ...initialData
    }
  });

  const { errors, isSubmitting } = formState;
  const [securityError, setSecurityError] = useState<string | null>(null);

  const onFormSubmit = async (data: UserFormValues) => {
    try {
      setSecurityError(null);
      
      // Security validation
      const securityCheck = await validateFormSecurity(data);
      if (!securityCheck.valid) {
        setSecurityError(securityCheck.error || 'Security validation failed');
        return;
      }

      await onSubmit(data);
    } catch (error) {
      setError('root', { 
        message: 'Failed to submit form. Please try again.' 
      });
    }
  };

  return (
    <form onSubmit={handleSubmit(onFormSubmit)} className="space-y-6">
      {securityError && (
        <div className="p-4 bg-red-50 border border-red-200 rounded-md">
          <p className="text-sm text-red-600">{securityError}</p>
        </div>
      )}

      <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
        <div>
          <label htmlFor="name" className="block text-sm font-medium text-gray-700 mb-1">
            Full Name *
          </label>
          <input
            {...register("name")}
            type="text"
            id="name"
            className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
            placeholder="Enter your full name"
          />
          {errors.name && (
            <p className="mt-1 text-sm text-red-600">{errors.name.message}</p>
          )}
        </div>

        <div>
          <label htmlFor="email" className="block text-sm font-medium text-gray-700 mb-1">
            Email Address *
          </label>
          <input
            {...register("email")}
            type="email"
            id="email"
            className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
            placeholder="your.email@example.com"
          />
          {errors.email && (
            <p className="mt-1 text-sm text-red-600">{errors.email.message}</p>
          )}
        </div>
      </div>

      <button
        type="submit"
        disabled={isSubmitting || loading}
        className="w-full bg-blue-600 text-white py-3 px-4 rounded-md hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-offset-2 disabled:opacity-50 disabled:cursor-not-allowed transition-colors"
      >
        {isSubmitting ? 'Submitting...' : 'Submit'}
      </button>
    </form>
  );
}
```

**Key Features Demonstrated:**
- Type-safe form handling with React Hook Form
- Zod schema validation with custom error messages
- Security validation integration
- Comprehensive error handling and user feedback
- Accessible form design with proper labels
- Loading states and disabled states

### 2. Custom Hook for Data Management

```typescript
// hooks/useDataManagement.ts
import { useState, useEffect, useCallback } from 'react';

interface DataItem {
  id: string;
  name: string;
  status: 'active' | 'inactive' | 'pending';
  createdAt: string;
  updatedAt: string;
}

interface UseDataManagementOptions {
  endpoint: string;
  initialFilters?: Record<string, any>;
  autoRefresh?: boolean;
  refreshInterval?: number;
}

interface UseDataManagementReturn {
  data: DataItem[];
  loading: boolean;
  error: string | null;
  filters: Record<string, any>;
  pagination: {
    page: number;
    totalPages: number;
    totalItems: number;
  };
  actions: {
    fetchData: (page?: number) => Promise<void>;
    updateItem: (id: string, updates: Partial<DataItem>) => Promise<void>;
    deleteItem: (id: string) => Promise<void>;
    setFilters: (filters: Record<string, any>) => void;
    refresh: () => Promise<void>;
  };
}

export function useDataManagement({
  endpoint,
  initialFilters = {},
  autoRefresh = false,
  refreshInterval = 30000
}: UseDataManagementOptions): UseDataManagementReturn {
  const [data, setData] = useState<DataItem[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const [filters, setFilters] = useState(initialFilters);
  const [pagination, setPagination] = useState({
    page: 1,
    totalPages: 1,
    totalItems: 0
  });

  const fetchData = useCallback(async (page: number = 1) => {
    setLoading(true);
    setError(null);

    try {
      const queryParams = new URLSearchParams({
        page: page.toString(),
        limit: '20',
        ...filters
      });

      const response = await fetch(`${endpoint}?${queryParams}`);
      
      if (!response.ok) {
        throw new Error(`HTTP error! status: ${response.status}`);
      }

      const result = await response.json();
      
      setData(result.items);
      setPagination({
        page: result.page,
        totalPages: result.totalPages,
        totalItems: result.totalItems
      });
    } catch (err) {
      setError(err instanceof Error ? err.message : 'An error occurred');
    } finally {
      setLoading(false);
    }
  }, [endpoint, filters]);

  const updateItem = useCallback(async (id: string, updates: Partial<DataItem>) => {
    try {
      const response = await fetch(`${endpoint}/${id}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(updates)
      });

      if (!response.ok) {
        throw new Error('Failed to update item');
      }

      // Optimistic update
      setData(prev => prev.map(item => 
        item.id === id ? { ...item, ...updates, updatedAt: new Date().toISOString() } : item
      ));
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Update failed');
      throw err;
    }
  }, [endpoint]);

  const deleteItem = useCallback(async (id: string) => {
    try {
      const response = await fetch(`${endpoint}/${id}`, {
        method: 'DELETE'
      });

      if (!response.ok) {
        throw new Error('Failed to delete item');
      }

      // Remove from local state
      setData(prev => prev.filter(item => item.id !== id));
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Delete failed');
      throw err;
    }
  }, [endpoint]);

  const refresh = useCallback(() => fetchData(pagination.page), [fetchData, pagination.page]);

  // Auto-refresh effect
  useEffect(() => {
    if (autoRefresh) {
      const interval = setInterval(refresh, refreshInterval);
      return () => clearInterval(interval);
    }
  }, [autoRefresh, refresh, refreshInterval]);

  // Fetch data when filters change
  useEffect(() => {
    fetchData(1);
  }, [fetchData]);

  return {
    data,
    loading,
    error,
    filters,
    pagination,
    actions: {
      fetchData,
      updateItem,
      deleteItem,
      setFilters,
      refresh
    }
  };
}
```

**Key Features Demonstrated:**
- Custom hook pattern for reusable logic
- TypeScript generics and interfaces
- Optimistic updates for better UX
- Error handling and loading states
- Auto-refresh functionality
- Pagination management

---

## Security Implementation

### 3. Security Middleware Class

```typescript
// lib/security/SecurityMiddleware.ts
import { NextRequest, NextResponse } from 'next/server';

interface SecurityResult {
  success: boolean;
  reason?: string;
  message?: string;
}

interface RateLimiterConfig {
  maxRequests: number;
  windowMs: number;
}

class RateLimiter {
  private requests = new Map<string, { count: number; resetTime: number }>();
  
  constructor(private config: RateLimiterConfig) {}

  isAllowed(identifier: string): boolean {
    const now = Date.now();
    const current = this.requests.get(identifier);

    if (!current || now > current.resetTime) {
      this.requests.set(identifier, {
        count: 1,
        resetTime: now + this.config.windowMs
      });
      return true;
    }

    if (current.count >= this.config.maxRequests) {
      return false;
    }

    current.count++;
    return true;
  }
}

export class SecurityMiddleware {
  private static rateLimiters = new Map<string, RateLimiter>();
  private static suspiciousIPs = new Set<string>();
  private static requestCounts = new Map<string, { count: number; lastReset: number }>();

  static initialize() {
    // Initialize rate limiters for different endpoint types
    this.rateLimiters.set('auth', new RateLimiter({ maxRequests: 5, windowMs: 15 * 60 * 1000 }));
    this.rateLimiters.set('api', new RateLimiter({ maxRequests: 100, windowMs: 60 * 1000 }));
    this.rateLimiters.set('general', new RateLimiter({ maxRequests: 1000, windowMs: 60 * 1000 }));
  }

  static getClientIP(request: NextRequest): string {
    const forwarded = request.headers.get('x-forwarded-for');
    const realIP = request.headers.get('x-real-ip');
    return forwarded?.split(',')[0] || realIP || 'unknown';
  }

  static isSuspiciousRequest(request: NextRequest): boolean {
    const url = request.url.toLowerCase();
    const userAgent = request.headers?.get('user-agent')?.toLowerCase() || '';

    // SQL injection patterns
    const sqlPatterns = [
      'union select', 'drop table', 'delete from', 'insert into', 'update set',
      'or 1=1', 'or 1=2', 'and 1=1', 'and 1=2', 'exec(', 'execute('
    ];

    // XSS patterns
    const xssPatterns = [
      'script>', 'javascript:', 'data:', 'vbscript:', 'onload=', 'onerror=',
      '<iframe', '<object', '<embed', 'document.cookie', 'window.location'
    ];

    // Path traversal patterns
    const pathTraversalPatterns = [
      '../', '..\\', '/etc/passwd', '/windows/system32', 'boot.ini'
    ];

    const allPatterns = [...sqlPatterns, ...xssPatterns, ...pathTraversalPatterns];
    const hasSuspiciousPattern = allPatterns.some(pattern => url.includes(pattern));

    // Check for suspicious user agents
    const suspiciousUserAgents = [
      'sqlmap', 'nikto', 'nmap', 'masscan', 'zap', 'burp', 'w3af'
    ];
    const hasSuspiciousUserAgent = suspiciousUserAgents.some(ua => userAgent.includes(ua));

    return hasSuspiciousPattern || hasSuspiciousUserAgent;
  }

  static async checkRateLimit(request: NextRequest, endpointType: string = 'general'): Promise<boolean> {
    const rateLimiter = this.rateLimiters.get(endpointType);
    if (!rateLimiter) return true;

    const clientIP = this.getClientIP(request);
    return rateLimiter.isAllowed(clientIP);
  }

  static async validateRequest(request: NextRequest, endpointType: string = 'general'): Promise<SecurityResult> {
    try {
      const clientIP = this.getClientIP(request);

      // Check if IP is flagged as suspicious
      if (this.suspiciousIPs.has(clientIP)) {
        this.logSecurityEvent('blocked_suspicious_ip', { clientIP });
        return { success: false, reason: 'suspicious_ip', message: 'Access denied' };
      }

      // Rate limiting check
      const rateLimitAllowed = await this.checkRateLimit(request, endpointType);
      if (!rateLimitAllowed) {
        this.logSecurityEvent('rate_limit_exceeded', { clientIP, endpoint: request.url });
        return { success: false, reason: 'rate_limited', message: 'Too many requests' };
      }

      // Threat detection
      if (this.isSuspiciousRequest(request)) {
        this.logSecurityEvent('suspicious_pattern', { 
          clientIP, 
          url: request.url,
          userAgent: request.headers?.get('user-agent')
        });
        this.suspiciousIPs.add(clientIP);
        return { success: false, reason: 'threat_detected', message: 'Suspicious request detected' };
      }

      return { success: true };
    } catch (error) {
      this.logSecurityEvent('security_check_error', { 
        error: error instanceof Error ? error.message : 'Unknown error'
      });
      return { success: false, reason: 'security_error', message: 'Security check failed' };
    }
  }

  private static logSecurityEvent(event: string, details: any) {
    const logEntry = {
      timestamp: new Date().toISOString(),
      event,
      details,
      severity: this.getSeverityLevel(event)
    };

    // In production, this would be sent to a security monitoring system
    console.warn('Security Event:', logEntry);
  }

  private static getSeverityLevel(event: string): 'low' | 'medium' | 'high' | 'critical' {
    const criticalEvents = ['suspicious_pattern', 'sql_injection_attempt'];
    const highEvents = ['rate_limit_exceeded', 'threat_detected'];
    const mediumEvents = ['security_check_error'];
    
    if (criticalEvents.includes(event)) return 'critical';
    if (highEvents.includes(event)) return 'high';
    if (mediumEvents.includes(event)) return 'medium';
    return 'low';
  }
}

// Initialize on module load
SecurityMiddleware.initialize();
```

**Security Features Demonstrated:**
- Multi-tier rate limiting system
- Advanced threat detection patterns
- IP-based security monitoring
- Comprehensive security logging
- Fail-secure error handling
- Configurable security policies

---

## Database & API Design

### 4. Secure API Route Handler

```typescript
// app/api/users/route.ts
import { NextRequest, NextResponse } from 'next/server';
import { z } from 'zod';
import { SecurityMiddleware } from '@/lib/security/SecurityMiddleware';
import { DatabaseSecurity } from '@/lib/security/DatabaseSecurity';

// Input validation schema
const CreateUserSchema = z.object({
  name: z.string().min(2).max(100),
  email: z.string().email(),
  role: z.enum(['admin', 'user', 'moderator']).default('user'),
  preferences: z.record(z.any()).optional()
});

const UpdateUserSchema = CreateUserSchema.partial();

// GET /api/users - Retrieve users with filtering and pagination
export async function GET(request: NextRequest) {
  try {
    // Security validation
    const securityResult = await SecurityMiddleware.validateRequest(request, 'api');
    if (!securityResult.success) {
      return NextResponse.json(
        { error: securityResult.message }, 
        { status: 429 }
      );
    }

    const { searchParams } = new URL(request.url);
    const page = parseInt(searchParams.get('page') || '1');
    const limit = parseInt(searchParams.get('limit') || '20');
    const search = searchParams.get('search') || '';
    const role = searchParams.get('role') || '';

    // Validate pagination parameters
    if (page < 1 || limit < 1 || limit > 100) {
      return NextResponse.json(
        { error: 'Invalid pagination parameters' }, 
        { status: 400 }
      );
    }

    // Fetch users with security wrapper
    const result = await DatabaseSecurity.secureQuery(
      async () => {
        const where: any = {};
        
        if (search) {
          where.OR = [
            { name: { contains: search, mode: 'insensitive' } },
            { email: { contains: search, mode: 'insensitive' } }
          ];
        }
        
        if (role) {
          where.role = role;
        }

        const [users, totalCount] = await Promise.all([
          prisma.user.findMany({
            where,
            orderBy: { createdAt: 'desc' },
            take: limit,
            skip: (page - 1) * limit,
            select: {
              id: true,
              name: true,
              email: true,
              role: true,
              createdAt: true,
              updatedAt: true
            }
          }),
          prisma.user.count({ where })
        ]);

        return {
          users,
          pagination: {
            page,
            limit,
            totalCount,
            totalPages: Math.ceil(totalCount / limit)
          }
        };
      },
      'get_users'
    );

    return NextResponse.json(result);
  } catch (error) {
    console.error('GET /api/users error:', error);
    return NextResponse.json(
      { error: 'Internal server error' }, 
      { status: 500 }
    );
  }
}

// POST /api/users - Create new user
export async function POST(request: NextRequest) {
  try {
    // Security validation
    const securityResult = await SecurityMiddleware.validateRequest(request, 'api');
    if (!securityResult.success) {
      return NextResponse.json(
        { error: securityResult.message }, 
        { status: 429 }
      );
    }

    // Parse and validate request body
    const body = await request.json();
    const validatedData = CreateUserSchema.parse(body);

    // Create user with security wrapper
    const result = await DatabaseSecurity.secureQuery(
      async () => {
        return await prisma.user.create({
          data: {
            ...validatedData,
            createdAt: new Date(),
            updatedAt: new Date()
          },
          select: {
            id: true,
            name: true,
            email: true,
            role: true,
            createdAt: true
          }
        });
      },
      'create_user'
    );

    return NextResponse.json(result, { status: 201 });
  } catch (error) {
    if (error instanceof z.ZodError) {
      return NextResponse.json(
        { error: 'Validation failed', details: error.errors }, 
        { status: 400 }
      );
    }

    console.error('POST /api/users error:', error);
    return NextResponse.json(
      { error: 'Internal server error' }, 
      { status: 500 }
    );
  }
}
```

**API Features Demonstrated:**
- Comprehensive security validation
- Input validation with Zod schemas
- Proper HTTP status codes
- Error handling and logging
- Pagination and filtering
- Database security wrapper
- Type-safe request/response handling

### 5. Database Security Layer

```typescript
// lib/security/DatabaseSecurity.ts
import { PrismaClient } from '@prisma/client';

interface QueryContext {
  operation: string;
  userId?: string;
  ip?: string;
  metadata?: Record<string, any>;
}

export class DatabaseSecurity {
  private static prisma: PrismaClient;
  private static queryLog: Array<{
    query: string;
    duration: number;
    timestamp: Date;
    context: string;
  }> = [];

  static initialize() {
    if (!this.prisma) {
      this.prisma = new PrismaClient({
        log: [
          { level: 'query', emit: 'event' },
          { level: 'error', emit: 'event' }
        ]
      });

      // Set up query logging
      this.prisma.$on('query', (e) => {
        this.logQuery(e.query, e.duration);
      });

      this.prisma.$on('error', (e) => {
        this.logSecurityEvent('database_error', {
          message: e.message,
          target: e.target
        });
      });
    }
  }

  static sanitizeInput(input: any): any {
    if (typeof input === 'string') {
      return input
        .replace(/['"\\]/g, '') // Remove quotes and backslashes
        .replace(/[;--]/g, '') // Remove SQL comment patterns
        .replace(/union\s+select/gi, '') // Remove UNION SELECT
        .replace(/drop\s+table/gi, '') // Remove DROP TABLE
        .replace(/delete\s+from/gi, '') // Remove DELETE FROM
        .trim();
    }
    
    if (Array.isArray(input)) {
      return input.map(item => this.sanitizeInput(item));
    }
    
    if (typeof input === 'object' && input !== null) {
      const sanitized: any = {};
      for (const [key, value] of Object.entries(input)) {
        sanitized[this.sanitizeInput(key)] = this.sanitizeInput(value);
      }
      return sanitized;
    }
    
    return input;
  }

  static validateQueryParams(params: any, context: string): boolean {
    try {
      const paramString = JSON.stringify(params).toLowerCase();
      
      const suspiciousPatterns = [
        'union select', 'drop table', 'delete from', 'insert into',
        'update set', 'exec(', 'execute(', 'xp_cmdshell'
      ];

      const hasSuspiciousPattern = suspiciousPatterns.some(pattern => 
        paramString.includes(pattern)
      );

      if (hasSuspiciousPattern) {
        this.logSecurityEvent('suspicious_query_params', {
          context,
          params: paramString
        });
        return false;
      }

      return true;
    } catch (error) {
      this.logSecurityEvent('query_validation_error', {
        context,
        error: error instanceof Error ? error.message : 'Unknown error'
      });
      return false;
    }
  }

  static async secureQuery<T>(
    operation: () => Promise<T>,
    context: string,
    queryContext?: QueryContext
  ): Promise<T> {
    const startTime = Date.now();
    
    try {
      // Validate context
      if (!this.validateQueryParams({ context }, context)) {
        throw new Error('Invalid query context');
      }

      // Execute operation
      const result = await operation();
      const duration = Date.now() - startTime;

      // Log successful query
      this.logQuery(`Operation: ${context}`, duration);

      // Check for performance issues
      if (duration > 5000) { // 5 seconds
        this.logSecurityEvent('slow_query', {
          context,
          duration,
          ...queryContext
        });
      }

      return result;
    } catch (error) {
      const duration = Date.now() - startTime;
      
      this.logSecurityEvent('database_operation_error', {
        context,
        error: error instanceof Error ? error.message : 'Unknown error',
        duration,
        ...queryContext
      });

      throw error;
    }
  }

  private static logQuery(query: string, duration: number) {
    const logEntry = {
      query: query.substring(0, 200), // Truncate long queries
      duration,
      timestamp: new Date()
    };

    this.queryLog.push(logEntry);

    // Keep only last 1000 queries in memory
    if (this.queryLog.length > 1000) {
      this.queryLog = this.queryLog.slice(-1000);
    }

    // Log slow queries
    if (duration > 1000) { // 1 second
      console.warn('Slow query detected:', logEntry);
    }
  }

  private static logSecurityEvent(event: string, details: any) {
    const logEntry = {
      event,
      details,
      timestamp: new Date().toISOString(),
      severity: this.getSeverityLevel(event)
    };

    console.warn('Database Security Event:', logEntry);
  }

  private static getSeverityLevel(event: string): 'low' | 'medium' | 'high' | 'critical' {
    const criticalEvents = ['suspicious_query_params', 'sql_injection_attempt'];
    const highEvents = ['database_operation_error', 'slow_query'];
    const mediumEvents = ['query_validation_error'];
    
    if (criticalEvents.includes(event)) return 'critical';
    if (highEvents.includes(event)) return 'high';
    if (mediumEvents.includes(event)) return 'medium';
    return 'low';
  }

  static getQueryStats() {
    const totalQueries = this.queryLog.length;
    const avgDuration = this.queryLog.reduce((sum, log) => sum + log.duration, 0) / totalQueries;
    const slowQueries = this.queryLog.filter(log => log.duration > 1000).length;
    
    return {
      totalQueries,
      avgDuration: Math.round(avgDuration),
      slowQueries
    };
  }
}

// Initialize on module load
DatabaseSecurity.initialize();
```

**Database Security Features:**
- Input sanitization and validation
- Query performance monitoring
- Comprehensive error logging
- SQL injection prevention
- Context-aware security checks
- Query statistics and monitoring

---

## Testing Strategies

### 6. Comprehensive Test Suite

```typescript
// __tests__/security/security-middleware.test.ts
import { SecurityMiddleware } from '@/lib/security/SecurityMiddleware';
import { NextRequest } from 'next/server';

describe('SecurityMiddleware', () => {
  beforeEach(() => {
    // Reset security state before each test
    SecurityMiddleware.initialize();
  });

  describe('Request Validation', () => {
    it('should allow legitimate requests', async () => {
      const request = new NextRequest('http://localhost:3000/api/users', {
        method: 'GET',
        headers: {
          'user-agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'
        }
      });

      const result = await SecurityMiddleware.validateRequest(request);
      expect(result.success).toBe(true);
    });

    it('should block SQL injection attempts', async () => {
      const request = new NextRequest('http://localhost:3000/api/users', {
        method: 'POST',
        body: JSON.stringify({ query: "'; DROP TABLE users; --" })
      });

      const result = await SecurityMiddleware.validateRequest(request);
      expect(result.success).toBe(false);
      expect(result.reason).toBe('threat_detected');
    });

    it('should block XSS attempts', async () => {
      const request = new NextRequest('http://localhost:3000/api/users?search=<script>alert("xss")</script>');

      const result = await SecurityMiddleware.validateRequest(request);
      expect(result.success).toBe(false);
      expect(result.reason).toBe('threat_detected');
    });

    it('should enforce rate limiting', async () => {
      const requests = Array(10).fill(null).map(() => 
        new NextRequest('http://localhost:3000/api/users')
      );

      const results = await Promise.all(
        requests.map(req => SecurityMiddleware.validateRequest(req, 'api'))
      );

      // Should allow some requests but block after limit
      const allowedCount = results.filter(r => r.success).length;
      expect(allowedCount).toBeLessThan(10);
    });
  });

  describe('IP Management', () => {
    it('should track suspicious IPs', async () => {
      const suspiciousRequest = new NextRequest('http://localhost:3000/api/users', {
        method: 'POST',
        body: JSON.stringify({ query: "'; DROP TABLE users; --" })
      });

      // First request should be blocked
      const result1 = await SecurityMiddleware.validateRequest(suspiciousRequest);
      expect(result1.success).toBe(false);

      // Second request from same IP should also be blocked
      const result2 = await SecurityMiddleware.validateRequest(suspiciousRequest);
      expect(result2.success).toBe(false);
      expect(result2.reason).toBe('suspicious_ip');
    });
  });
});

// __tests__/integration/api-endpoints.test.ts
import { createMocks } from 'node-mocks-http';
import handler from '@/app/api/users/route';

describe('/api/users', () => {
  describe('GET /api/users', () => {
    it('should return paginated users', async () => {
      const { req, res } = createMocks({
        method: 'GET',
        url: '/api/users?page=1&limit=10'
      });

      await handler(req, res);

      expect(res._getStatusCode()).toBe(200);
      const data = JSON.parse(res._getData());
      expect(data).toHaveProperty('users');
      expect(data).toHaveProperty('pagination');
      expect(data.pagination).toHaveProperty('page', 1);
      expect(data.pagination).toHaveProperty('limit', 10);
    });

    it('should handle invalid pagination parameters', async () => {
      const { req, res } = createMocks({
        method: 'GET',
        url: '/api/users?page=-1&limit=1000'
      });

      await handler(req, res);

      expect(res._getStatusCode()).toBe(400);
      const data = JSON.parse(res._getData());
      expect(data).toHaveProperty('error');
    });
  });

  describe('POST /api/users', () => {
    it('should create new user with valid input', async () => {
      const { req, res } = createMocks({
        method: 'POST',
        body: {
          name: 'John Doe',
          email: 'john@example.com',
          role: 'user'
        }
      });

      await handler(req, res);

      expect(res._getStatusCode()).toBe(201);
      const data = JSON.parse(res._getData());
      expect(data).toHaveProperty('id');
      expect(data).toHaveProperty('name', 'John Doe');
    });

    it('should reject invalid input', async () => {
      const { req, res } = createMocks({
        method: 'POST',
        body: {
          name: 'A', // Too short
          email: 'invalid-email', // Invalid email
          role: 'invalid' // Invalid role
        }
      });

      await handler(req, res);

      expect(res._getStatusCode()).toBe(400);
      const data = JSON.parse(res._getData());
      expect(data).toHaveProperty('error');
    });
  });
});

// __tests__/components/UserForm.test.tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import UserForm from '@/components/forms/UserForm';

describe('UserForm', () => {
  const mockOnSubmit = jest.fn();

  beforeEach(() => {
    mockOnSubmit.mockClear();
  });

  it('should render form fields correctly', () => {
    render(<UserForm onSubmit={mockOnSubmit} />);

    expect(screen.getByLabelText(/full name/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/email address/i)).toBeInTheDocument();
    expect(screen.getByRole('button', { name: /submit/i })).toBeInTheDocument();
  });

  it('should validate required fields', async () => {
    const user = userEvent.setup();
    render(<UserForm onSubmit={mockOnSubmit} />);

    const submitButton = screen.getByRole('button', { name: /submit/i });
    await user.click(submitButton);

    await waitFor(() => {
      expect(screen.getByText(/name must be at least 2 characters/i)).toBeInTheDocument();
      expect(screen.getByText(/invalid email format/i)).toBeInTheDocument();
    });

    expect(mockOnSubmit).not.toHaveBeenCalled();
  });

  it('should submit form with valid data', async () => {
    const user = userEvent.setup();
    render(<UserForm onSubmit={mockOnSubmit} />);

    await user.type(screen.getByLabelText(/full name/i), 'John Doe');
    await user.type(screen.getByLabelText(/email address/i), 'john@example.com');
    await user.selectOptions(screen.getByLabelText(/role/i), 'user');

    const submitButton = screen.getByRole('button', { name: /submit/i });
    await user.click(submitButton);

    await waitFor(() => {
      expect(mockOnSubmit).toHaveBeenCalledWith({
        name: 'John Doe',
        email: 'john@example.com',
        role: 'user',
        preferences: {
          notifications: true,
          theme: 'light'
        }
      });
    });
  });

  it('should handle submission errors', async () => {
    const user = userEvent.setup();
    mockOnSubmit.mockRejectedValueOnce(new Error('Submission failed'));
    
    render(<UserForm onSubmit={mockOnSubmit} />);

    await user.type(screen.getByLabelText(/full name/i), 'John Doe');
    await user.type(screen.getByLabelText(/email address/i), 'john@example.com');
    await user.click(screen.getByRole('button', { name: /submit/i }));

    await waitFor(() => {
      expect(screen.getByText(/failed to submit form/i)).toBeInTheDocument();
    });
  });
});
```

**Testing Features Demonstrated:**
- Unit tests for security middleware
- Integration tests for API endpoints
- Component tests with user interactions
- Mock implementations and test utilities
- Error handling and edge case testing
- Comprehensive test coverage

---

## Performance Optimization

### 7. Performance Monitoring Hook

```typescript
// hooks/usePerformanceMonitoring.ts
import { useEffect, useRef, useState } from 'react';

interface PerformanceMetrics {
  renderTime: number;
  apiResponseTime: number;
  memoryUsage: number;
  errorCount: number;
}

interface UsePerformanceMonitoringOptions {
  componentName: string;
  trackApiCalls?: boolean;
  trackMemoryUsage?: boolean;
  logToConsole?: boolean;
}

export function usePerformanceMonitoring({
  componentName,
  trackApiCalls = true,
  trackMemoryUsage = true,
  logToConsole = false
}: UsePerformanceMonitoringOptions) {
  const [metrics, setMetrics] = useState<PerformanceMetrics>({
    renderTime: 0,
    apiResponseTime: 0,
    memoryUsage: 0,
    errorCount: 0
  });

  const renderStartTime = useRef<number>(0);
  const apiStartTimes = useRef<Map<string, number>>(new Map());

  // Track component render time
  useEffect(() => {
    renderStartTime.current = performance.now();
    
    return () => {
      const renderTime = performance.now() - renderStartTime.current;
      setMetrics(prev => ({ ...prev, renderTime }));
      
      if (logToConsole) {
        console.log(`${componentName} render time: ${renderTime.toFixed(2)}ms`);
      }
    };
  }, [componentName, logToConsole]);

  // Track memory usage
  useEffect(() => {
    if (!trackMemoryUsage || !('memory' in performance)) return;

    const updateMemoryUsage = () => {
      const memory = (performance as any).memory;
      if (memory) {
        const memoryUsage = memory.usedJSHeapSize / 1024 / 1024; // MB
        setMetrics(prev => ({ ...prev, memoryUsage }));
      }
    };

    const interval = setInterval(updateMemoryUsage, 5000);
    return () => clearInterval(interval);
  }, [trackMemoryUsage]);

  // API call tracking
  const trackApiCall = (apiName: string) => {
    if (!trackApiCalls) return;
    apiStartTimes.current.set(apiName, performance.now());
  };

  const endApiCall = (apiName: string) => {
    if (!trackApiCalls) return;
    
    const startTime = apiStartTimes.current.get(apiName);
    if (startTime) {
      const responseTime = performance.now() - startTime;
      setMetrics(prev => ({ ...prev, apiResponseTime: responseTime }));
      apiStartTimes.current.delete(apiName);
      
      if (logToConsole) {
        console.log(`${apiName} response time: ${responseTime.toFixed(2)}ms`);
      }
    }
  };

  const trackError = () => {
    setMetrics(prev => ({ ...prev, errorCount: prev.errorCount + 1 }));
  };

  return {
    metrics,
    trackApiCall,
    endApiCall,
    trackError
  };
}

// Usage example in a component
export function UserList() {
  const { metrics, trackApiCall, endApiCall, trackError } = usePerformanceMonitoring({
    componentName: 'UserList',
    logToConsole: process.env.NODE_ENV === 'development'
  });

  const fetchUsers = async () => {
    try {
      trackApiCall('fetchUsers');
      const response = await fetch('/api/users');
      const data = await response.json();
      endApiCall('fetchUsers');
      return data;
    } catch (error) {
      trackError();
      endApiCall('fetchUsers');
      throw error;
    }
  };

  return (
    <div>
      {/* Component content */}
      {process.env.NODE_ENV === 'development' && (
        <div className="fixed bottom-4 right-4 bg-black text-white p-2 rounded text-xs">
          <div>Render: {metrics.renderTime.toFixed(2)}ms</div>
          <div>API: {metrics.apiResponseTime.toFixed(2)}ms</div>
          <div>Memory: {metrics.memoryUsage.toFixed(2)}MB</div>
          <div>Errors: {metrics.errorCount}</div>
        </div>
      )}
    </div>
  );
}
```

**Performance Features Demonstrated:**
- Component render time tracking
- API response time monitoring
- Memory usage tracking
- Error counting and monitoring
- Development-only performance overlay
- Custom performance monitoring hooks

---

## Summary

These code snippets demonstrate:

✅ **Advanced TypeScript Usage**: Generics, interfaces, type safety  
✅ **Modern React Patterns**: Hooks, context, custom components  
✅ **Security Implementation**: Multi-layer security approach  
✅ **Database Design**: Secure database operations and monitoring  
✅ **API Design**: RESTful endpoints with proper validation  
✅ **Testing Strategies**: Comprehensive test coverage  
✅ **Performance Optimization**: Monitoring and optimization techniques  
✅ **Error Handling**: Robust error handling and user feedback  
✅ **Code Quality**: Clean, maintainable, and well-documented code  

Each snippet is designed to showcase technical skills while maintaining privacy and not revealing any proprietary business logic or sensitive information.
