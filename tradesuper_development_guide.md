"expr": "rate(orders_total[5m])",
            "legendFormat": "Total Orders/sec"
          },
          {
            "expr": "rate(orders_total{order_type=\"B2B\"}[5m])",
            "legendFormat": "B2B Orders/sec"
          },
          {
            "expr": "rate(orders_total{order_type=\"B2C\"}[5m])",
            "legendFormat": "B2C Orders/sec"
          }
        ],
        "gridPos": {"h": 8, "w": 24, "x": 0, "y": 8}
      },
      {
        "id": 4,
        "title": "Commission Tracking",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(increase(commission_earned_total[1h]))",
            "legendFormat": "Total Commission (1h)"
          },
          {
            "expr": "sum(increase(commission_earned_total{commission_type=\"B2C\"}[1h]))",
            "legendFormat": "B2C Commission (7%)"
          },
          {
            "expr": "sum(increase(commission_earned_total{commission_type=\"B2B\"}[1h]))",
            "legendFormat": "B2B Commission (3%)"
          }
        ],
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 16}
      },
      {
        "id": 5,
        "title": "Loyalty Points Activity",
        "type": "graph",
        "targets": [
          {
            "expr": "rate(loyalty_points_earned_total[5m])",
            "legendFormat": "Points Earned/sec"
          },
          {
            "expr": "rate(loyalty_points_redeemed_total[5m])",
            "legendFormat": "Points Redeemed/sec"
          }
        ],
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 16}
      },
      {
        "id": 6,
        "title": "Inventory Status",
        "type": "table",
        "targets": [
          {
            "expr": "product_stock_quantity < product_min_stock_level",
            "legendFormat": "Low Stock Products",
            "format": "table"
          }
        ],
        "gridPos": {"h": 8, "w": 24, "x": 0, "y": 24}
      },
      {
        "id": 7,
        "title": "Top Products by Revenue",
        "type": "bargauge",
        "targets": [
          {
            "expr": "topk(10, sum by (product_name) (increase(product_revenue_total[24h])))",
            "legendFormat": "{{ product_name }}"
          }
        ],
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 32}
      },
      {
        "id": 8,
        "title": "User Activity",
        "type": "graph",
        "targets": [
          {
            "expr": "sum by (user_role) (active_users_total)",
            "legendFormat": "{{ user_role }}"
          }
        ],
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 32}
      },
      {
        "id": 9,
        "title": "Promotion Effectiveness",
        "type": "table",
        "targets": [
          {
            "expr": "sum by (promotion_name) (increase(promotion_usage_total[24h]))",
            "legendFormat": "{{ promotion_name }}",
            "format": "table"
          }
        ],
        "gridPos": {"h": 8, "w": 24, "x": 0, "y": 40}
      }
    ],
    "time": {"from": "now-24h", "to": "now"},
    "refresh": "30s"
  }
}
```

### **Step 27: Application Health Checks & SLOs**

```javascript
// Backend health check implementation (routes/health.js)
const express = require('express');
const router = express.Router();
const { sequelize } = require('../config/database');
const redis = require('../config/redis');
const AWS = require('aws-sdk');

// Health check metrics
let healthMetrics = {
  startTime: new Date(),
  totalRequests: 0,
  healthyChecks: 0,
  unhealthyChecks: 0,
  lastCheck: null,
  components: {
    database: { status: 'unknown', lastCheck: null, responseTime: 0 },
    redis: { status: 'unknown', lastCheck: null, responseTime: 0 },
    aws: { status: 'unknown', lastCheck: null, responseTime: 0 },
    memory: { status: 'unknown', usage: 0, limit: 0 },
    cpu: { status: 'unknown', usage: 0 }
  }
};

// Basic health check
router.get('/', async (req, res) => {
  healthMetrics.totalRequests++;
  
  const health = {
    status: 'healthy',
    timestamp: new Date().toISOString(),
    uptime: Math.floor((Date.now() - healthMetrics.startTime.getTime()) / 1000),
    version: process.env.APP_VERSION || '1.0.0',
    environment: process.env.NODE_ENV || 'development'
  };

  try {
    // Quick database ping
    await sequelize.authenticate();
    healthMetrics.healthyChecks++;
    res.status(200).json(health);
  } catch (error) {
    healthMetrics.unhealthyChecks++;
    health.status = 'unhealthy';
    health.error = 'Database connection failed';
    res.status(503).json(health);
  }
});

// Detailed health check
router.get('/detailed', async (req, res) => {
  const startTime = Date.now();
  
  try {
    // Check database
    const dbStart = Date.now();
    await sequelize.authenticate();
    await sequelize.query('SELECT 1');
    healthMetrics.components.database = {
      status: 'healthy',
      lastCheck: new Date(),
      responseTime: Date.now() - dbStart
    };
  } catch (error) {
    healthMetrics.components.database = {
      status: 'unhealthy',
      lastCheck: new Date(),
      error: error.message,
      responseTime: Date.now() - dbStart
    };
  }

  try {
    // Check Redis
    const redisStart = Date.now();
    await redis.ping();
    healthMetrics.components.redis = {
      status: 'healthy',
      lastCheck: new Date(),
      responseTime: Date.now() - redisStart
    };
  } catch (error) {
    healthMetrics.components.redis = {
      status: 'unhealthy',
      lastCheck: new Date(),
      error: error.message,
      responseTime: Date.now() - redisStart
    };
  }

  try {
    // Check AWS connectivity
    const awsStart = Date.now();
    const s3 = new AWS.S3();
    await s3.headBucket({ Bucket: process.env.AWS_S3_BUCKET }).promise();
    healthMetrics.components.aws = {
      status: 'healthy',
      lastCheck: new Date(),
      responseTime: Date.now() - awsStart
    };
  } catch (error) {
    healthMetrics.components.aws = {
      status: 'unhealthy',
      lastCheck: new Date(),
      error: error.message,
      responseTime: Date.now() - awsStart
    };
  }

  // Check memory usage
  const memUsage = process.memoryUsage();
  const memLimit = 1024 * 1024 * 1024; // 1GB limit
  healthMetrics.components.memory = {
    status: memUsage.heapUsed < memLimit * 0.9 ? 'healthy' : 'warning',
    usage: memUsage.heapUsed,
    limit: memLimit,
    percentage: Math.round((memUsage.heapUsed / memLimit) * 100)
  };

  // Determine overall health
  const componentStatuses = Object.values(healthMetrics.components).map(c => c.status);
  const isHealthy = componentStatuses.every(status => status === 'healthy');
  const hasWarnings = componentStatuses.some(status => status === 'warning');

  const overallStatus = isHealthy ? 'healthy' : hasWarnings ? 'warning' : 'unhealthy';
  const responseTime = Date.now() - startTime;

  healthMetrics.lastCheck = new Date();

  const health = {
    status: overallStatus,
    timestamp: new Date().toISOString(),
    responseTime: `${responseTime}ms`,
    uptime: Math.floor((Date.now() - healthMetrics.startTime.getTime()) / 1000),
    version: process.env.APP_VERSION || '1.0.0',
    environment: process.env.NODE_ENV || 'development',
    components: healthMetrics.components,
    metrics: {
      totalRequests: healthMetrics.totalRequests,
      healthyChecks: healthMetrics.healthyChecks,
      unhealthyChecks: healthMetrics.unhealthyChecks,
      successRate: healthMetrics.totalRequests > 0 
        ? Math.round((healthMetrics.healthyChecks / healthMetrics.totalRequests) * 100) 
        : 0
    }
  };

  const statusCode = overallStatus === 'healthy' ? 200 : 
                    overallStatus === 'warning' ? 200 : 503;

  if (overallStatus === 'healthy') {
    healthMetrics.healthyChecks++;
  } else {
    healthMetrics.unhealthyChecks++;
  }

  res.status(statusCode).json(health);
});

// Readiness check (for Kubernetes)
router.get('/ready', async (req, res) => {
  try {
    // Check if application is ready to serve traffic
    await sequelize.authenticate();
    
    // Check if migrations are complete
    const [results] = await sequelize.query(
      "SELECT EXISTS (SELECT 1 FROM information_schema.tables WHERE table_name = 'users')"
    );
    
    if (!results[0].exists) {
      throw new Error('Database not properly initialized');
    }

    res.status(200).json({
      status: 'ready',
      timestamp: new Date().toISOString()
    });
  } catch (error) {
    res.status(503).json({
      status: 'not ready',
      error: error.message,
      timestamp: new Date().toISOString()
    });
  }
});

// Liveness check (for Kubernetes)
router.get('/live', (req, res) => {
  // Simple check that the process is running
  res.status(200).json({
    status: 'alive',
    timestamp: new Date().toISOString(),
    pid: process.pid,
    uptime: process.uptime()
  });
});

// Metrics endpoint for Prometheus
router.get('/metrics', (req, res) => {
  const metrics = `
# HELP tradesuper_health_check_total Total health checks performed
# TYPE tradesuper_health_check_total counter
tradesuper_health_check_total{result="success"} ${healthMetrics.healthyChecks}
tradesuper_health_check_total{result="failure"} ${healthMetrics.unhealthyChecks}

# HELP tradesuper_health_check_duration_seconds Health check response time
# TYPE tradesuper_health_check_duration_seconds histogram
${Object.entries(healthMetrics.components).map(([name, component]) => 
  `tradesuper_component_response_time_seconds{component="${name}"} ${component.responseTime / 1000}`
).join('\n')}

# HELP tradesuper_memory_usage_bytes Memory usage in bytes
# TYPE tradesuper_memory_usage_bytes gauge
tradesuper_memory_usage_bytes ${healthMetrics.components.memory.usage || 0}

# HELP tradesuper_uptime_seconds Application uptime
# TYPE tradesuper_uptime_seconds counter
tradesuper_uptime_seconds ${Math.floor((Date.now() - healthMetrics.startTime.getTime()) / 1000)}
`;

  res.set('Content-Type', 'text/plain');
  res.send(metrics);
});

module.exports = router;

// Business metrics endpoint (routes/business-metrics.js)
const express = require('express');
const router = express.Router();
const { Order, Product, User, LoyaltyTransaction } = require('../models');
const { Op } = require('sequelize');

// Business metrics for monitoring
router.get('/metrics', async (req, res) => {
  try {
    const now = new Date();
    const oneHourAgo = new Date(now.getTime() - 60 * 60 * 1000);
    const oneDayAgo = new Date(now.getTime() - 24 * 60 * 60 * 1000);

    // Order metrics
    const totalOrders = await Order.count();
    const ordersLastHour = await Order.count({
      where: { createdAt: { [Op.gte]: oneHourAgo } }
    });
    const ordersLast24h = await Order.count({
      where: { createdAt: { [Op.gte]: oneDayAgo } }
    });

    const b2bOrders = await Order.count({
      where: { orderType: 'B2B', createdAt: { [Op.gte]: oneDayAgo } }
    });
    const b2cOrders = await Order.count({
      where: { orderType: 'B2C', createdAt: { [Op.gte]: oneDayAgo } }
    });

    // Revenue metrics
    const totalRevenue = await Order.sum('totalAmount', {
      where: { status: { [Op.ne]: 'cancelled' } }
    }) || 0;
    
    const revenueLastHour = await Order.sum('totalAmount', {
      where: { 
        createdAt: { [Op.gte]: oneHourAgo },
        status: { [Op.ne]: 'cancelled' }
      }
    }) || 0;

    const b2bRevenue = await Order.sum('totalAmount', {
      where: { 
        orderType: 'B2B',
        createdAt: { [Op.gte]: oneDayAgo },
        status: { [Op.ne]: 'cancelled' }
      }
    }) || 0;

    const b2cRevenue = await Order.sum('totalAmount', {
      where: { 
        orderType: 'B2C',
        createdAt: { [Op.gte]: oneDayAgo },
        status: { [Op.ne]: 'cancelled' }
      }
    }) || 0;

    // Commission metrics
    const totalCommission = await Order.sum('commissionAmount', {
      where: { status: { [Op.ne]: 'cancelled' } }
    }) || 0;

    const commissionLastHour = await Order.sum('commissionAmount', {
      where: { 
        createdAt: { [Op.gte]: oneHourAgo },
        status: { [Op.ne]: 'cancelled' }
      }
    }) || 0;

    // User metrics
    const totalUsers = await User.count();
    const activeUsers = await User.count({
      where: { 
        lastLogin: { [Op.gte]: oneDayAgo },
        status: 'active'
      }
    });

    const usersByRole = await User.findAll({
      attributes: ['role', [sequelize.fn('COUNT', sequelize.col('id')), 'count']],
      group: ['role'],
      raw: true
    });

    // Product metrics
    const totalProducts = await Product.count({ where: { isActive: true } });
    const lowStockProducts = await Product.count({
      where: { 
        stockQuantity: { [Op.lte]: sequelize.col('minStockLevel') },
        isActive: true
      }
    });
    const outOfStockProducts = await Product.count({
      where: { stockQuantity: 0, isActive: true }
    });

    // Loyalty metrics
    const loyaltyPointsEarned = await LoyaltyTransaction.sum('points', {
      where: { 
        transactionType: 'earned',
        createdAt: { [Op.gte]: oneDayAgo }
      }
    }) || 0;

    const loyaltyPointsRedeemed = await LoyaltyTransaction.sum('points', {
      where: { 
        transactionType: 'spent',
        createdAt: { [Op.gte]: oneDayAgo }
      }
    }) || 0;

    // Format as Prometheus metrics
    const metrics = `
# HELP tradesuper_orders_total Total number of orders
# TYPE tradesuper_orders_total counter
tradesuper_orders_total ${totalOrders}
tradesuper_orders_total{timeframe="1h"} ${ordersLastHour}
tradesuper_orders_total{timeframe="24h"} ${ordersLast24h}
tradesuper_orders_total{type="B2B",timeframe="24h"} ${b2bOrders}
tradesuper_orders_total{type="B2C",timeframe="24h"} ${b2cOrders}

# HELP tradesuper_revenue_total Total revenue in dollars
# TYPE tradesuper_revenue_total counter
tradesuper_revenue_total ${totalRevenue}
tradesuper_revenue_total{timeframe="1h"} ${revenueLastHour}
tradesuper_revenue_total{type="B2B",timeframe="24h"} ${b2bRevenue}
tradesuper_revenue_total{type="B2C",timeframe="24h"} ${b2cRevenue}

# HELP tradesuper_commission_total Total commission earned
# TYPE tradesuper_commission_total counter
tradesuper_commission_total ${totalCommission}
tradesuper_commission_total{timeframe="1h"} ${commissionLastHour}

# HELP tradesuper_users_total Total number of users
# TYPE tradesuper_users_total gauge
tradesuper_users_total ${totalUsers}
tradesuper_users_active_total{timeframe="24h"} ${activeUsers}
${usersByRole.map(user => 
  `tradesuper_users_total{role="${user.role}"} ${user.count}`
).join('\n')}

# HELP tradesuper_products_total Total number of products
# TYPE tradesuper_products_total gauge
tradesuper_products_total ${totalProducts}
tradesuper_products_low_stock_total ${lowStockProducts}
tradesuper_products_out_of_stock_total ${outOfStockProducts}

# HELP tradesuper_loyalty_points_total Loyalty points activity
# TYPE tradesuper_loyalty_points_total counter
tradesuper_loyalty_points_earned_total{timeframe="24h"} ${loyaltyPointsEarned}
tradesuper_loyalty_points_redeemed_total{timeframe="24h"} ${loyaltyPointsRedeemed}

# HELP tradesuper_metrics_last_updated_timestamp Last time metrics were updated
# TYPE tradesuper_metrics_last_updated_timestamp gauge
tradesuper_metrics_last_updated_timestamp ${Math.floor(Date.now() / 1000)}
`;

    res.set('Content-Type', 'text/plain');
    res.send(metrics);
  } catch (error) {
    console.error('Business metrics error:', error);
    res.status(500).json({ error: 'Failed to generate business metrics' });
  }
});

module.exports = router;
```

---

## 🎯 **Complete Implementation Summary**

### **✅ FULLY IMPLEMENTED FEATURES:**

#### **🔐 Backend Core (Node.js + PostgreSQL + Redis)**
- **Authentication & Authorization:** JWT-based auth with role-based access control
- **User Management:** Owner, Admin, Supermarket, Customer roles with different permissions
- **Product Management:** Full CRUD with B2B/B2C pricing, inventory tracking, categories
- **Advanced Cart System:** Real-time updates, price calculations, stock validation
- **Order Processing:** Complete order lifecycle with status tracking, commission calculation
- **Promotions System:** Individual products, categories, recurring events (Ramadan, Friday deals)
- **Coupon System:** Code-based discounts with validation, usage limits, expiration
- **Loyalty Points:** Earn/spend system with tier levels, transaction history, expiration
- **Notification System:** Push notifications, email alerts, in-app notifications, preferences
- **Reorder Feature:** Save order templates, quick reorder from history, auto-reorder
- **Wishlist & Reviews:** Product favorites, ratings, verified purchase reviews
- **Advanced Analytics:** Revenue tracking, commission reports, user activity, inventory alerts

#### **📱 Frontend Core (Flutter - iOS/Android)**
- **Multi-Role Interfaces:** Distinct dashboards for Owner, Admin, Supermarket, Customer
- **Advanced Product Catalog:** Search, filters, categories, pagination, image galleries
- **Smart Cart Management:** Real-time updates, coupon application, loyalty points usage
- **Complete Order Flow:** Checkout, payment integration, order tracking, reorder
- **Promotion Display:** Featured deals, flash sales, seasonal offers, countdown timers
- **Loyalty Dashboard:** Points balance, tier progress, transaction history, redemption
- **Notification Center:** Push notifications, in-app alerts, notification preferences
- **Offline Capability:** Local storage, sync when online, cached data
- **Multi-Language:** Arabic and English support with RTL layout

#### **🚀 DevOps & Production (Docker + Kubernetes + AWS)**
- **Containerization:** Production-ready Docker images with multi-stage builds
- **Orchestration:** Kubernetes deployment with auto-scaling, health checks, monitoring
- **Infrastructure:** AWS EKS, RDS PostgreSQL, ElastiCache Redis, S3, CloudFront
- **CI/CD Pipeline:** GitHub Actions with automated testing, security scanning, deployment
- **Monitoring Stack:** Prometheus, Grafana, AlertManager with business metrics
- **Security:** Rate limiting, input validation, encryption, secret management
- **Performance:** Load balancing, caching, CDN, database optimization

#### **🧪 Testing & Quality (Comprehensive Test Suite)**
- **Backend Testing:** Unit tests, integration tests, API testing, database testing
- **Frontend Testing:** Widget tests, integration tests, performance tests, UI tests
- **End-to-End Testing:** Complete user journeys, business flow validation
- **Performance Testing:** Load testing, memory usage, response time monitoring
- **Security Testing:** Vulnerability scanning, penetration testing, compliance checks

### **🔥 ADVANCED FEATURES INCLUDED:**

#### **💼 Business Intelligence**
- **Real-time Analytics:** Live dashboards with business metrics, revenue tracking
- **Commission Tracking:** Automatic calculation (7% B2C, 3% B2B) with detailed reporting
- **Inventory Management:** Stock alerts, low stock notifications, reorder suggestions
- **Customer Insights:** Purchase patterns, loyalty analysis, lifetime value tracking
- **Admin Tools:** User management, bulk operations, data export, reporting

#### **🎯 Marketing & Promotions**
- **Event-Based Promotions:** Ramadan specials, Friday deals, seasonal offers
- **Smart Coupons:** Usage limits, minimum order amounts, category restrictions
- **Loyalty Tiers:** Bronze, Silver, Gold, Platinum, Diamond with increasing benefits
- **Personalization:** Recommended products, targeted offers, purchase history analysis

#### **⚡ Performance & Scalability**
- **Auto-Scaling:** Horizontal pod autoscaling based on CPU/memory usage
- **Caching Strategy:** Redis caching, CDN for images, database query optimization
- **Load Balancing:** Multiple replicas with intelligent traffic distribution
- **Performance Monitoring:** APM integration, response time tracking, bottleneck identification

### **📋 DEPLOYMENT CHECKLIST:**

1. **✅ Set up AWS infrastructure using Terraform**
2. **✅ Deploy Kubernetes cluster with monitoring**
3. **✅ Configure CI/CD pipeline with GitHub Actions**
4. **✅ Set up database with proper migrations**
5. **✅ Configure Redis for session/cache management**
6. **✅ Set up S3 and CloudFront for image hosting**
7. **✅ Configure monitoring and alerting**
8. **✅ Deploy applications with health checks**
9. **✅ Run end-to-end tests**
10. **✅ Monitor production metrics**

### **🚀 READY FOR PRODUCTION:**

This comprehensive guide provides everything needed to build, test, and deploy your TradeSuper platform. The implementation covers all requested features including advanced discounts, loyalty systems, coupons, notifications, cart functionality, reorder capabilities, and recurring promotions.

**The platform is designed to:**
- Handle high traffic with auto-scaling
- Provide real-time business insights
- Support complex promotional campaigns
- Maintain data consistency and security
- Offer excellent user experience across all roles
- Scale efficiently as your business grows

You now have a complete, production-ready blueprint for your B2B/B2C wholesale-retail platform! 🎉/aws"
  
  cluster_name    = "tradesuper-${var.environment}"
  cluster_version = "1.28"
  
  vpc_id                         = module.vpc.vpc_id
  subnet_ids                     = module.vpc.private_subnets
  cluster_endpoint_public_access = true
  
  # EKS Managed Node Groups
  eks_managed_node_groups = {
    main = {
      name = "tradesuper-nodes"
      
      instance_types = ["t3.medium"]
      min_size     = 3
      max_size     = 10
      desired_size = 5
      
      disk_size = 50
      
      labels = {
        Environment = var.environment
        NodeGroup   = "main"
      }
      
      tags = {
        Environment = var.environment
        Project     = "tradesuper"
      }
    }
  }
  
  tags = {
    Environment = var.environment
    Project     = "tradesuper"
  }
}

# RDS PostgreSQL
resource "aws_db_subnet_group" "tradesuper" {
  name       = "tradesuper-${var.environment}"
  subnet_ids = module.vpc.private_subnets
  
  tags = {
    Name        = "TradeSuper DB subnet group"
    Environment = var.environment
  }
}

resource "aws_security_group" "rds" {
  name_prefix = "tradesuper-rds-${var.environment}"
  vpc_id      = module.vpc.vpc_id
  
  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [module.eks.cluster_security_group_id]
  }
  
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  tags = {
    Name        = "tradesuper-rds-${var.environment}"
    Environment = var.environment
  }
}

resource "aws_db_instance" "tradesuper" {
  identifier = "tradesuper-${var.environment}"
  
  engine         = "postgres"
  engine_version = "15.4"
  instance_class = var.environment == "production" ? "db.t3.large" : "db.t3.micro"
  
  allocated_storage     = var.environment == "production" ? 100 : 20
  max_allocated_storage = var.environment == "production" ? 1000 : 100
  storage_type          = "gp2"
  storage_encrypted     = true
  
  db_name  = "tradesuper"
  username = "tradesuper_user"
  password = var.db_password
  
  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.tradesuper.name
  
  backup_retention_period = var.environment == "production" ? 30 : 7
  backup_window          = "03:00-04:00"
  maintenance_window     = "sun:04:00-sun:05:00"
  
  skip_final_snapshot = var.environment != "production"
  deletion_protection = var.environment == "production"
  
  performance_insights_enabled = var.environment == "production"
  monitoring_interval         = var.environment == "production" ? 60 : 0
  
  tags = {
    Name        = "tradesuper-${var.environment}"
    Environment = var.environment
  }
}

# ElastiCache Redis
resource "aws_elasticache_subnet_group" "tradesuper" {
  name       = "tradesuper-${var.environment}"
  subnet_ids = module.vpc.private_subnets
}

resource "aws_security_group" "redis" {
  name_prefix = "tradesuper-redis-${var.environment}"
  vpc_id      = module.vpc.vpc_id
  
  ingress {
    from_port       = 6379
    to_port         = 6379
    protocol        = "tcp"
    security_groups = [module.eks.cluster_security_group_id]
  }
  
  tags = {
    Name        = "tradesuper-redis-${var.environment}"
    Environment = var.environment
  }
}

resource "aws_elasticache_replication_group" "tradesuper" {
  replication_group_id       = "tradesuper-${var.environment}"
  description                = "Redis for TradeSuper ${var.environment}"
  
  node_type                  = var.environment == "production" ? "cache.t3.medium" : "cache.t3.micro"
  port                       = 6379
  parameter_group_name       = "default.redis7"
  
  num_cache_clusters         = var.environment == "production" ? 3 : 1
  automatic_failover_enabled = var.environment == "production"
  multi_az_enabled          = var.environment == "production"
  
  subnet_group_name = aws_elasticache_subnet_group.tradesuper.name
  security_group_ids = [aws_security_group.redis.id]
  
  at_rest_encryption_enabled = true
  transit_encryption_enabled = true
  
  tags = {
    Name        = "tradesuper-${var.environment}"
    Environment = var.environment
  }
}

# S3 Buckets
resource "aws_s3_bucket" "tradesuper_images" {
  bucket = "tradesuper-images-${var.environment}-${random_string.bucket_suffix.result}"
  
  tags = {
    Name        = "TradeSuper Images"
    Environment = var.environment
  }
}

resource "aws_s3_bucket_versioning" "tradesuper_images" {
  bucket = aws_s3_bucket.tradesuper_images.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_encryption" "tradesuper_images" {
  bucket = aws_s3_bucket.tradesuper_images.id
  
  server_side_encryption_configuration {
    rule {
      apply_server_side_encryption_by_default {
        sse_algorithm = "AES256"
      }
    }
  }
}

resource "aws_s3_bucket_public_access_block" "tradesuper_images" {
  bucket = aws_s3_bucket.tradesuper_images.id
  
  block_public_acls       = false
  block_public_policy     = false
  ignore_public_acls      = false
  restrict_public_buckets = false
}

resource "aws_s3_bucket_cors_configuration" "tradesuper_images" {
  bucket = aws_s3_bucket.tradesuper_images.id
  
  cors_rule {
    allowed_headers = ["*"]
    allowed_methods = ["GET", "PUT", "POST"]
    allowed_origins = ["*"]
    expose_headers  = ["ETag"]
    max_age_seconds = 3000
  }
}

# CloudFront Distribution
resource "aws_cloudfront_distribution" "tradesuper_images" {
  origin {
    domain_name = aws_s3_bucket.tradesuper_images.bucket_regional_domain_name
    origin_id   = "S3-${aws_s3_bucket.tradesuper_images.id}"
    
    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.tradesuper.cloudfront_access_identity_path
    }
  }
  
  enabled             = true
  is_ipv6_enabled     = true
  default_root_object = "index.html"
  
  default_cache_behavior {
    allowed_methods        = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "S3-${aws_s3_bucket.tradesuper_images.id}"
    compress               = true
    viewer_protocol_policy = "redirect-to-https"
    
    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }
    
    min_ttl     = 0
    default_ttl = 3600
    max_ttl     = 86400
  }
  
  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }
  
  viewer_certificate {
    cloudfront_default_certificate = true
  }
  
  tags = {
    Name        = "tradesuper-images-${var.environment}"
    Environment = var.environment
  }
}

resource "aws_cloudfront_origin_access_identity" "tradesuper" {
  comment = "TradeSuper ${var.environment} OAI"
}

# Random string for unique bucket naming
resource "random_string" "bucket_suffix" {
  length  = 8
  special = false
  upper   = false
}

# Variables
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "db_password" {
  description = "Database password"
  type        = string
  sensitive   = true
}

# Outputs
output "cluster_name" {
  description = "EKS cluster name"
  value       = module.eks.cluster_name
}

output "cluster_endpoint" {
  description = "EKS cluster endpoint"
  value       = module.eks.cluster_endpoint
}

output "rds_endpoint" {
  description = "RDS instance endpoint"
  value       = aws_db_instance.tradesuper.endpoint
}

output "redis_endpoint" {
  description = "Redis cluster endpoint"
  value       = aws_elasticache_replication_group.tradesuper.primary_endpoint_address
}

output "s3_bucket" {
  description = "S3 bucket name for images"
  value       = aws_s3_bucket.tradesuper_images.id
}

output "cloudfront_domain" {
  description = "CloudFront distribution domain"
  value       = aws_cloudfront_distribution.tradesuper_images.domain_name
}
```

### **Step 25: Helm Chart for Application Deployment**

```yaml
# helm/tradesuper/Chart.yaml
apiVersion: v2
name: tradesuper
description: TradeSuper B2B/B2C Platform
type: application
version: 1.0.0
appVersion: "1.0.0"

dependencies:
  - name: postgresql
    version: 12.8.2
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
  - name: redis
    version: 17.11.3
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled

# helm/tradesuper/values.yaml
global:
  imageRegistry: ""
  storageClass: ""

environment: production

image:
  backend:
    repository: tradesuper/backend
    tag: "latest"
    pullPolicy: IfNotPresent
  frontend:
    repository: tradesuper/frontend
    tag: "latest"
    pullPolicy: IfNotPresent

imagePullSecrets: []

backend:
  replicaCount: 3
  
  service:
    type: ClusterIP
    port: 80
    targetPort: 3000
  
  resources:
    limits:
      cpu: 500m
      memory: 1Gi
    requests:
      cpu: 250m
      memory: 512Mi
  
  autoscaling:
    enabled: true
    minReplicas: 3
    maxReplicas: 10
    targetCPUUtilizationPercentage: 70
    targetMemoryUtilizationPercentage: 80
  
  env:
    NODE_ENV: production
    B2C_COMMISSION_RATE: "0.07"
    B2B_COMMISSION_RATE: "0.03"
  
  secrets:
    - name: DB_HOST
      valueFrom:
        secretKeyRef:
          name: tradesuper-secrets
          key: db-host
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: tradesuper-secrets
          key: db-password
    - name: JWT_SECRET
      valueFrom:
        secretKeyRef:
          name: tradesuper-secrets
          key: jwt-secret

frontend:
  replicaCount: 2
  
  service:
    type: ClusterIP
    port: 80
    targetPort: 80
  
  resources:
    limits:
      cpu: 200m
      memory: 256Mi
    requests:
      cpu: 100m
      memory: 128Mi

cronjobs:
  expirePoints:
    enabled: true
    schedule: "0 2 * * *"
    image:
      repository: tradesuper/backend
      tag: "latest"
    command: ["node", "scripts/expirePoints.js"]
    
  activatePromotions:
    enabled: true
    schedule: "0 * * * *"
    image:
      repository: tradesuper/backend
      tag: "latest"
    command: ["node", "scripts/activatePromotions.js"]
    
  generateAnalytics:
    enabled: true
    schedule: "0 1 * * *"
    image:
      repository: tradesuper/backend
      tag: "latest"
    command: ["node", "scripts/generateAnalytics.js"]

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/rate-limit: "1000"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
  hosts:
    - host: api.tradesuper.com
      paths:
        - path: /
          pathType: Prefix
          service: backend
    - host: app.tradesuper.com
      paths:
        - path: /
          pathType: Prefix
          service: frontend
  tls:
    - secretName: tradesuper-tls
      hosts:
        - api.tradesuper.com
        - app.tradesuper.com

postgresql:
  enabled: false  # Using external RDS
  auth:
    postgresPassword: ""
    username: "tradesuper_user"
    password: ""
    database: "tradesuper"

redis:
  enabled: false  # Using external ElastiCache
  auth:
    enabled: false

monitoring:
  enabled: true
  serviceMonitor:
    enabled: true
    labels:
      release: prometheus
  grafana:
    dashboards:
      enabled: true

# helm/tradesuper/templates/backend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "tradesuper.fullname" . }}-backend
  labels:
    {{- include "tradesuper.labels" . | nindent 4 }}
    component: backend
spec:
  {{- if not .Values.backend.autoscaling.enabled }}
  replicas: {{ .Values.backend.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "tradesuper.selectorLabels" . | nindent 6 }}
      component: backend
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
      labels:
        {{- include "tradesuper.selectorLabels" . | nindent 8 }}
        component: backend
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      initContainers:
      - name: wait-for-db
        image: busybox:1.28
        command: ['sh', '-c']
        args:
        - |
          until nc -z {{ .Values.database.host }} {{ .Values.database.port }}; do
            echo "Waiting for database..."
            sleep 2
          done
          echo "Database is ready!"
      containers:
      - name: backend
        image: "{{ .Values.image.backend.repository }}:{{ .Values.image.backend.tag }}"
        imagePullPolicy: {{ .Values.image.backend.pullPolicy }}
        ports:
        - name: http
          containerPort: 3000
          protocol: TCP
        env:
        {{- range $key, $value := .Values.backend.env }}
        - name: {{ $key }}
          value: {{ $value | quote }}
        {{- end }}
        {{- range .Values.backend.secrets }}
        - name: {{ .name }}
          {{- toYaml .valueFrom | nindent 10 }}
        {{- end }}
        livenessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3
        startupProbe:
          httpGet:
            path: /health
            port: http
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 30
        resources:
          {{- toYaml .Values.backend.resources | nindent 12 }}

# helm/tradesuper/templates/cronjob-expire-points.yaml
{{- if .Values.cronjobs.expirePoints.enabled }}
apiVersion: batch/v1
kind: CronJob
metadata:
  name: {{ include "tradesuper.fullname" . }}-expire-points
  labels:
    {{- include "tradesuper.labels" . | nindent 4 }}
    component: cronjob
spec:
  schedule: {{ .Values.cronjobs.expirePoints.schedule | quote }}
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 1
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            {{- include "tradesuper.selectorLabels" . | nindent 12 }}
            component: cronjob
        spec:
          restartPolicy: OnFailure
          containers:
          - name: expire-points
            image: "{{ .Values.cronjobs.expirePoints.image.repository }}:{{ .Values.cronjobs.expirePoints.image.tag }}"
            command: {{ .Values.cronjobs.expirePoints.command }}
            env:
            {{- range $key, $value := .Values.backend.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
            {{- range .Values.backend.secrets }}
            - name: {{ .name }}
              {{- toYaml .valueFrom | nindent 14 }}
            {{- end }}
            resources:
              limits:
                cpu: 200m
                memory: 256Mi
              requests:
                cpu: 100m
                memory: 128Mi
{{- end }}
```

## 5.2 Advanced Monitoring & Alerting

### **Step 26: Comprehensive Monitoring Stack**

```yaml
# monitoring/prometheus-values.yaml
prometheus:
  prometheusSpec:
    retention: 30d
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp2
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi
    
    additionalScrapeConfigs:
    - job_name: 'tradesuper-backend'
      kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names:
          - tradesuper
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_name]
        action: keep
        regex: tradesuper-backend-service
      - source_labels: [__meta_kubernetes_endpoint_port_name]
        action: keep
        regex: metrics
      
    - job_name: 'business-metrics'
      kubernetes_sd_configs:
      - role: endpoints
        namespaces:
          names:
          - tradesuper
      relabel_configs:
      - source_labels: [__meta_kubernetes_service_name]
        action: keep
        regex: tradesuper-backend-service
      - source_labels: [__meta_kubernetes_endpoint_port_name]
        action: keep
        regex: business-metrics

    ruleFiles:
    - /etc/prometheus/rules/tradesuper.rules.yml

alertmanager:
  config:
    global:
      slack_api_url: 'YOUR_SLACK_WEBHOOK_URL'
      smtp_smarthost: 'smtp.gmail.com:587'
      smtp_from: 'alerts@tradesuper.com'
    
    route:
      group_by: ['alertname', 'cluster', 'service']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 12h
      receiver: 'default'
      routes:
      - match:
          severity: critical
        receiver: 'critical-alerts'
      - match:
          severity: warning
        receiver: 'warning-alerts'
      - match:
          alertname: 'BusinessMetricAnomaly'
        receiver: 'business-alerts'
    
    receivers:
    - name: 'default'
      slack_configs:
      - channel: '#alerts'
        title: 'TradeSuper Alert'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
    
    - name: 'critical-alerts'
      slack_configs:
      - channel: '#alerts-critical'
        title: 'CRITICAL: {{ .GroupLabels.alertname }}'
        text: |
          {{ range .Alerts }}
          *Alert:* {{ .Annotations.summary }}
          *Description:* {{ .Annotations.description }}
          *Severity:* {{ .Labels.severity }}
          {{ end }}
        send_resolved: true
      email_configs:
      - to: 'oncall@tradesuper.com'
        subject: 'Critical Alert: {{ .GroupLabels.alertname }}'
        body: |
          {{ range .Alerts }}
          Alert: {{ .Annotations.summary }}
          Description: {{ .Annotations.description }}
          Severity: {{ .Labels.severity }}
          Time: {{ .StartsAt }}
          {{ end }}
    
    - name: 'warning-alerts'
      slack_configs:
      - channel: '#alerts-warning'
        title: 'Warning: {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
        send_resolved: true
    
    - name: 'business-alerts'
      slack_configs:
      - channel: '#business-alerts'
        title: 'Business Alert: {{ .GroupLabels.alertname }}'
        text: |
          {{ range .Alerts }}
          *Metric:* {{ .Labels.metric }}
          *Value:* {{ .Labels.value }}
          *Description:* {{ .Annotations.description }}
          {{ end }}

# monitoring/business-rules.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tradesuper-business-rules
  namespace: monitoring
data:
  tradesuper-business.rules.yml: |
    groups:
    - name: tradesuper.business
      rules:
      # Order Processing Alerts
      - alert: HighOrderFailureRate
        expr: rate(orders_failed_total[5m]) / rate(orders_total[5m]) > 0.05
        for: 2m
        labels:
          severity: critical
          category: business
        annotations:
          summary: "High order failure rate detected"
          description: "Order failure rate is {{ $value | humanizePercentage }} over the last 5 minutes"
      
      - alert: LowOrderVolume
        expr: rate(orders_total[1h]) < 10
        for: 15m
        labels:
          severity: warning
          category: business
        annotations:
          summary: "Low order volume detected"
          description: "Only {{ $value }} orders processed in the last hour"
      
      - alert: OrderValueAnomaly
        expr: |
          (
            avg_over_time(order_value_total[1h]) 
            - 
            avg_over_time(order_value_total[1h] offset 24h)
          ) / avg_over_time(order_value_total[1h] offset 24h) > 0.5
        for: 10m
        labels:
          severity: warning
          category: business
        annotations:
          summary: "Unusual order value pattern detected"
          description: "Average order value has changed by {{ $value | humanizePercentage }} compared to same time yesterday"
      
      # Revenue & Commission Alerts
      - alert: CommissionCalculationError
        expr: increase(commission_calculation_errors_total[5m]) > 0
        for: 1m
        labels:
          severity: critical
          category: business
        annotations:
          summary: "Commission calculation errors detected"
          description: "{{ $value }} commission calculation errors in the last 5 minutes"
      
      - alert: RevenueDropAlert
        expr: |
          (
            sum(rate(revenue_total[1h])) 
            - 
            sum(rate(revenue_total[1h] offset 24h))
          ) / sum(rate(revenue_total[1h] offset 24h)) < -0.3
        for: 15m
        labels:
          severity: warning
          category: business
        annotations:
          summary: "Significant revenue drop detected"
          description: "Revenue has dropped by {{ $value | humanizePercentage }} compared to yesterday"
      
      # Inventory & Stock Alerts
      - alert: CriticalStockLevels
        expr: |
          sum by (product_id, product_name) (
            product_stock_quantity{stock_quantity < min_stock_level}
          ) > 0
        for: 5m
        labels:
          severity: warning
          category: inventory
        annotations:
          summary: "Products below minimum stock level"
          description: "Product {{ $labels.product_name }} (ID: {{ $labels.product_id }}) is below minimum stock level"
      
      - alert: OutOfStockProducts
        expr: |
          sum by (product_id, product_name) (
            product_stock_quantity{stock_quantity = 0}
          ) > 0
        for: 1m
        labels:
          severity: critical
          category: inventory
        annotations:
          summary: "Products out of stock"
          description: "Product {{ $labels.product_name }} (ID: {{ $labels.product_id }}) is out of stock"
      
      # User & Authentication Alerts
      - alert: HighLoginFailureRate
        expr: rate(login_failures_total[5m]) > 10
        for: 2m
        labels:
          severity: warning
          category: security
        annotations:
          summary: "High login failure rate detected"
          description: "{{ $value }} login failures per second over the last 5 minutes"
      
      - alert: SuspiciousUserActivity
        expr: |
          sum by (user_id) (
            increase(orders_total[10m])
          ) > 50
        for: 5m
        labels:
          severity: warning
          category: security
        annotations:
          summary: "Suspicious user activity detected"
          description: "User {{ $labels.user_id }} has placed {{ $value }} orders in 10 minutes"
      
      # Loyalty & Promotion Alerts
      - alert: LoyaltyPointsAnomalousActivity
        expr: increase(loyalty_points_earned_total[1h]) > 100000
        for: 10m
        labels:
          severity: warning
          category: business
        annotations:
          summary: "Unusual loyalty points activity"
          description: "{{ $value }} loyalty points earned in the last hour - investigation required"
      
      - alert: PromotionUsageSpike
        expr: |
          rate(promotion_usage_total[10m]) > 
          3 * rate(promotion_usage_total[10m] offset 1h)
        for: 5m
        labels:
          severity: warning
          category: business
        annotations:
          summary: "Promotion usage spike detected"
          description: "Promotion usage is {{ $value }}x higher than normal - potential abuse"
      
      # Payment & Transaction Alerts
      - alert: PaymentFailureSpike
        expr: rate(payment_failures_total[5m]) > 1
        for: 2m
        labels:
          severity: critical
          category: business
        annotations:
          summary: "Payment failure spike detected"
          description: "{{ $value }} payment failures per second - check payment gateway"
      
      - alert: RefundRateHigh
        expr: |
          rate(refunds_total[1h]) / rate(orders_total[1h]) > 0.1
        for: 10m
        labels:
          severity: warning
          category: business
        annotations:
          summary: "High refund rate detected"
          description: "Refund rate is {{ $value | humanizePercentage }} - investigate quality issues"

# monitoring/grafana-dashboard-business.json
{
  "dashboard": {
    "id": null,
    "title": "TradeSuper Business Metrics",
    "tags": ["tradesuper", "business"],
    "timezone": "browser",
    "panels": [
      {
        "id": 1,
        "title": "Order Overview",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(increase(orders_total[1h]))",
            "legendFormat": "Orders (1h)"
          },
          {
            "expr": "sum(increase(revenue_total[1h]))",
            "legendFormat": "Revenue (1h)"
          },
          {
            "expr": "sum(orders_total) - sum(orders_total offset 24h)",
            "legendFormat": "Orders vs Yesterday"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "color": {"mode": "palette-classic"},
            "unit": "short",
            "decimals": 0
          }
        },
        "gridPos": {"h": 8, "w": 12, "x": 0, "y": 0}
      },
      {
        "id": 2,
        "title": "Revenue by Order Type",
        "type": "piechart",
        "targets": [
          {
            "expr": "sum by (order_type) (increase(revenue_total[24h]))",
            "legendFormat": "{{ order_type }}"
          }
        ],
        "gridPos": {"h": 8, "w": 12, "x": 12, "y": 0}
      },
      {
        "id": 3,
        "title": "Order Rate",
        "type": "graph",
        "targets": [
          {
             DateTime.now(),
      updatedAt: DateTime.now(),
    );
  }

  static Cart createMockCart({
    int id = 1,
    int userId = 1,
    List<CartItem>? items,
    double subtotal = 50.0,
    double total = 50.0,
  }) {
    return Cart(
      id: id,
      userId: userId,
      items: items ?? [createMockCartItem()],
      subtotal: subtotal,
      total: total,
      itemCount: items?.length ?? 1,
      updatedAt: DateTime.now(),
    );
  }

  static CartItem createMockCartItem({
    int id = 1,
    int cartId = 1,
    int productId = 1,
    Product? product,
    int quantity = 2,
    String priceType = 'B2C',
    double unitPrice = 25.0,
  }) {
    return CartItem(
      id: id,
      cartId: cartId,
      productId: productId,
      product: product ?? createMockProduct(),
      quantity: quantity,
      priceType: priceType,
      unitPrice: unitPrice,
      createdAt: DateTime.now(),
      updatedAt: DateTime.now(),
    );
  }

  static Order createMockOrder({
    int id = 1,
    String orderNumber = 'ORD-123456',
    int buyerId = 1,
    int adminId = 2,
    OrderType orderType = OrderType.b2c,
    double totalAmount = 50.0,
    OrderStatus status = OrderStatus.pending,
  }) {
    return Order(
      id: id,
      orderNumber: orderNumber,
      buyerId: buyerId,
      adminId: adminId,
      orderType: orderType,
      subtotal: totalAmount,
      totalAmount: totalAmount,
      commissionRate: 0.07,
      commissionAmount: totalAmount * 0.07,
      status: status,
      paymentStatus: PaymentStatus.pending,
      createdAt: DateTime.now(),
      updatedAt: DateTime.now(),
    );
  }

  // Widget testing helpers
  static Widget createTestWidget({
    required Widget child,
    List<ChangeNotifierProvider>? providers,
  }) {
    return MaterialApp(
      home: providers != null
          ? MultiProvider(
              providers: providers,
              child: child,
            )
          : child,
    );
  }

  static List<ChangeNotifierProvider> createMockProviders({
    AuthProvider? authProvider,
    CartProvider? cartProvider,
    ProductProvider? productProvider,
  }) {
    return [
      if (authProvider != null)
        ChangeNotifierProvider<AuthProvider>.value(value: authProvider),
      if (cartProvider != null)
        ChangeNotifierProvider<CartProvider>.value(value: cartProvider),
      if (productProvider != null)
        ChangeNotifierProvider<ProductProvider>.value(value: productProvider),
    ];
  }

  // Common test actions
  static Future<void> tapButtonByText(WidgetTester tester, String text) async {
    await tester.tap(find.text(text));
    await tester.pumpAndSettle();
  }

  static Future<void> tapButtonByKey(WidgetTester tester, Key key) async {
    await tester.tap(find.byKey(key));
    await tester.pumpAndSettle();
  }

  static Future<void> enterText(WidgetTester tester, Key fieldKey, String text) async {
    await tester.enterText(find.byKey(fieldKey), text);
    await tester.pumpAndSettle();
  }

  static Future<void> scrollToWidget(WidgetTester tester, Finder finder) async {
    await tester.ensureVisible(finder);
    await tester.pumpAndSettle();
  }

  // Assertion helpers
  static void expectWidgetExists(Finder finder) {
    expect(finder, findsOneWidget);
  }

  static void expectWidgetNotExists(Finder finder) {
    expect(finder, findsNothing);
  }

  static void expectTextExists(String text) {
    expect(find.text(text), findsOneWidget);
  }

  static void expectLoadingIndicator() {
    expect(find.byType(CircularProgressIndicator), findsOneWidget);
  }

  static void expectErrorMessage(String message) {
    expect(find.text(message), findsOneWidget);
  }
}

// Mock classes
class MockAuthProvider extends Mock implements AuthProvider {}
class MockCartProvider extends Mock implements CartProvider {}
class MockProductProvider extends Mock implements ProductProvider {}
class MockOrderProvider extends Mock implements OrderProvider {}
class MockLoyaltyProvider extends Mock implements LoyaltyProvider {}

// test/unit/providers/auth_provider_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/mockito.dart';
import 'package:dartz/dartz.dart';
import 'package:tradesuper_app/presentation/providers/auth_provider.dart';
import 'package:tradesuper_app/data/repositories/auth_repository.dart';
import 'package:tradesuper_app/core/services/storage_service.dart';
import 'package:tradesuper_app/core/errors/failures.dart';
import '../../helpers/test_helpers.dart';

class MockAuthRepository extends Mock implements AuthRepository {}
class MockStorageService extends Mock implements StorageService {}

void main() {
  group('AuthProvider', () {
    late AuthProvider authProvider;
    late MockAuthRepository mockAuthRepository;
    late MockStorageService mockStorageService;

    setUp(() {
      mockAuthRepository = MockAuthRepository();
      mockStorageService = MockStorageService();
      authProvider = AuthProvider(mockAuthRepository, mockStorageService);
    });

    group('login', () {
      test('should authenticate user successfully', () async {
        // Arrange
        final mockUser = TestHelpers.createMockUser();
        final mockAuthResponse = AuthResponse(
          user: mockUser,
          token: 'test_token',
        );

        when(mockAuthRepository.login(
          email: anyNamed('email'),
          password: anyNamed('password'),
        )).thenAnswer((_) async => Right(mockAuthResponse));

        when(mockStorageService.saveAuthToken(any))
            .thenAnswer((_) async => true);
        when(mockStorageService.saveUserRole(any))
            .thenAnswer((_) async => true);

        // Act
        final result = await authProvider.login(
          email: 'test@example.com',
          password: 'password123',
        );

        // Assert
        expect(result, true);
        expect(authProvider.isAuthenticated, true);
        expect(authProvider.currentUser, mockUser);
        expect(authProvider.error, null);
        
        verify(mockStorageService.saveAuthToken('test_token')).called(1);
        verify(mockStorageService.saveUserRole('customer')).called(1);
      });

      test('should handle authentication failure', () async {
        // Arrange
        when(mockAuthRepository.login(
          email: anyNamed('email'),
          password: anyNamed('password'),
        )).thenAnswer((_) async => Left(ServerFailure('Invalid credentials')));

        // Act
        final result = await authProvider.login(
          email: 'test@example.com',
          password: 'wrongpassword',
        );

        // Assert
        expect(result, false);
        expect(authProvider.isAuthenticated, false);
        expect(authProvider.currentUser, null);
        expect(authProvider.error, 'Invalid credentials');
        
        verifyNever(mockStorageService.saveAuthToken(any));
      });

      test('should handle network error', () async {
        // Arrange
        when(mockAuthRepository.login(
          email: anyNamed('email'),
          password: anyNamed('password'),
        )).thenThrow(Exception('Network error'));

        // Act
        final result = await authProvider.login(
          email: 'test@example.com',
          password: 'password123',
        );

        // Assert
        expect(result, false);
        expect(authProvider.error, 'An unexpected error occurred');
      });
    });

    group('logout', () {
      test('should clear user data and tokens', () async {
        // Arrange
        authProvider.currentUser = TestHelpers.createMockUser();
        when(mockAuthRepository.logout()).thenAnswer((_) async => Right(null));
        when(mockStorageService.clearAuthToken()).thenAnswer((_) async => true);
        when(mockStorageService.clearUserRole()).thenAnswer((_) async => true);

        // Act
        await authProvider.logout();

        // Assert
        expect(authProvider.isAuthenticated, false);
        expect(authProvider.currentUser, null);
        
        verify(mockStorageService.clearAuthToken()).called(1);
        verify(mockStorageService.clearUserRole()).called(1);
      });
    });

    group('register', () => {
      test('should register new user successfully', () async {
        // Arrange
        final mockUser = TestHelpers.createMockUser();
        final mockAuthResponse = AuthResponse(
          user: mockUser,
          token: 'test_token',
        );

        when(mockAuthRepository.register(
          email: anyNamed('email'),
          password: anyNamed('password'),
          fullName: anyNamed('fullName'),
          role: anyNamed('role'),
        )).thenAnswer((_) async => Right(mockAuthResponse));

        when(mockStorageService.saveAuthToken(any))
            .thenAnswer((_) async => true);

        // Act
        final result = await authProvider.register(
          email: 'newuser@example.com',
          password: 'password123',
          fullName: 'New User',
        );

        // Assert
        expect(result, true);
        expect(authProvider.currentUser, mockUser);
        expect(authProvider.isAuthenticated, true);
      });
    });
  });
}

// test/widget/screens/cart_screen_test.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:mockito/mockito.dart';
import 'package:provider/provider.dart';
import 'package:tradesuper_app/presentation/screens/customer/cart_screen.dart';
import 'package:tradesuper_app/presentation/providers/cart_provider.dart';
import '../../helpers/test_helpers.dart';

void main() {
  group('CartScreen Widget Tests', () => {
    late MockCartProvider mockCartProvider;

    setUp(() {
      mockCartProvider = MockCartProvider();
    });

    testWidgets('should show empty cart message when cart is empty', (tester) async {
      // Arrange
      when(mockCartProvider.isLoading).thenReturn(false);
      when(mockCartProvider.hasItems).thenReturn(false);
      when(mockCartProvider.cart).thenReturn(null);
      when(mockCartProvider.error).thenReturn(null);

      // Act
      await tester.pumpWidget(
        TestHelpers.createTestWidget(
          child: CartScreen(),
          providers: [
            ChangeNotifierProvider<CartProvider>.value(value: mockCartProvider),
          ],
        ),
      );

      // Assert
      TestHelpers.expectTextExists('Your cart is empty');
      TestHelpers.expectTextExists('Add some products to get started');
      TestHelpers.expectWidgetExists(find.byIcon(Icons.shopping_cart_outlined));
      TestHelpers.expectWidgetExists(find.text('Start Shopping'));
    });

    testWidgets('should show loading indicator when loading', (tester) async {
      // Arrange
      when(mockCartProvider.isLoading).thenReturn(true);
      when(mockCartProvider.hasItems).thenReturn(false);
      when(mockCartProvider.cart).thenReturn(null);
      when(mockCartProvider.error).thenReturn(null);

      // Act
      await tester.pumpWidget(
        TestHelpers.createTestWidget(
          child: CartScreen(),
          providers: [
            ChangeNotifierProvider<CartProvider>.value(value: mockCartProvider),
          ],
        ),
      );

      // Assert
      TestHelpers.expectLoadingIndicator();
    });

    testWidgets('should show error message when error occurs', (tester) async {
      // Arrange
      const errorMessage = 'Failed to load cart';
      when(mockCartProvider.isLoading).thenReturn(false);
      when(mockCartProvider.hasItems).thenReturn(false);
      when(mockCartProvider.cart).thenReturn(null);
      when(mockCartProvider.error).thenReturn(errorMessage);

      // Act
      await tester.pumpWidget(
        TestHelpers.createTestWidget(
          child: CartScreen(),
          providers: [
            ChangeNotifierProvider<CartProvider>.value(value: mockCartProvider),
          ],
        ),
      );

      // Assert
      TestHelpers.expectErrorMessage(errorMessage);
      TestHelpers.expectWidgetExists(find.text('Retry'));
    });

    testWidgets('should show cart items when cart has items', (tester) async {
      // Arrange
      final mockCart = TestHelpers.createMockCart();
      when(mockCartProvider.isLoading).thenReturn(false);
      when(mockCartProvider.hasItems).thenReturn(true);
      when(mockCartProvider.cart).thenReturn(mockCart);
      when(mockCartProvider.error).thenReturn(null);
      when(mockCartProvider.subtotal).thenReturn(50.0);
      when(mockCartProvider.total).thenReturn(50.0);

      // Act
      await tester.pumpWidget(
        TestHelpers.createTestWidget(
          child: CartScreen(),
          providers: [
            ChangeNotifierProvider<CartProvider>.value(value: mockCartProvider),
          ],
        ),
      );

      // Assert
      TestHelpers.expectTextExists('Test Product');
      TestHelpers.expectWidgetExists(find.text('Proceed to Checkout'));
    });

    testWidgets('should call removeItem when remove button is tapped', (tester) async {
      // Arrange
      final mockCart = TestHelpers.createMockCart();
      when(mockCartProvider.isLoading).thenReturn(false);
      when(mockCartProvider.hasItems).thenReturn(true);
      when(mockCartProvider.cart).thenReturn(mockCart);
      when(mockCartProvider.error).thenReturn(null);
      when(mockCartProvider.removeItem(any)).thenAnswer((_) async => true);

      // Act
      await tester.pumpWidget(
        TestHelpers.createTestWidget(
          child: CartScreen(),
          providers: [
            ChangeNotifierProvider<CartProvider>.value(value: mockCartProvider),
          ],
        ),
      );

      await tester.tap(find.byIcon(Icons.delete));
      await tester.pumpAndSettle();

      // Assert
      verify(mockCartProvider.removeItem(any)).called(1);
    });

    testWidgets('should show clear cart confirmation dialog', (tester) async {
      // Arrange
      final mockCart = TestHelpers.createMockCart();
      when(mockCartProvider.isLoading).thenReturn(false);
      when(mockCartProvider.hasItems).thenReturn(true);
      when(mockCartProvider.cart).thenReturn(mockCart);
      when(mockCartProvider.error).thenReturn(null);

      // Act
      await tester.pumpWidget(
        TestHelpers.createTestWidget(
          child: CartScreen(),
          providers: [
            ChangeNotifierProvider<CartProvider>.value(value: mockCartProvider),
          ],
        ),
      );

      await tester.tap(find.text('Clear All'));
      await tester.pumpAndSettle();

      // Assert
      TestHelpers.expectTextExists('Clear Cart');
      TestHelpers.expectTextExists('Are you sure you want to remove all items from your cart?');
      TestHelpers.expectWidgetExists(find.text('Cancel'));
      TestHelpers.expectWidgetExists(find.text('Clear'));
    });

    testWidgets('should call clearCart when confirmed', (tester) async {
      // Arrange
      final mockCart = TestHelpers.createMockCart();
      when(mockCartProvider.isLoading).thenReturn(false);
      when(mockCartProvider.hasItems).thenReturn(true);
      when(mockCartProvider.cart).thenReturn(mockCart);
      when(mockCartProvider.error).thenReturn(null);
      when(mockCartProvider.clearCart()).thenAnswer((_) async => true);

      // Act
      await tester.pumpWidget(
        TestHelpers.createTestWidget(
          child: CartScreen(),
          providers: [
            ChangeNotifierProvider<CartProvider>.value(value: mockCartProvider),
          ],
        ),
      );

      await tester.tap(find.text('Clear All'));
      await tester.pumpAndSettle();
      
      await tester.tap(find.text('Clear'));
      await tester.pumpAndSettle();

      // Assert
      verify(mockCartProvider.clearCart()).called(1);
    });
  });
}

// test/integration/app_test.dart (Integration Tests)
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:tradesuper_app/main.dart' as app;

void main() {
  IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  group('TradeSuper App Integration Tests', () {
    testWidgets('complete customer journey - login, browse, add to cart, checkout', (tester) async {
      // Start the app
      app.main();
      await tester.pumpAndSettle();

      // Wait for splash screen to finish
      await tester.pumpAndSettle(Duration(seconds: 3));

      // Test login flow
      await tester.tap(find.byKey(Key('customer_login_button')));
      await tester.pumpAndSettle();

      await tester.enterText(find.byKey(Key('email_field')), 'customer@test.com');
      await tester.enterText(find.byKey(Key('password_field')), 'password123');
      
      await tester.tap(find.byKey(Key('login_submit_button')));
      await tester.pumpAndSettle(Duration(seconds: 3));

      // Verify dashboard is shown
      expect(find.text('Welcome back!'), findsOneWidget);

      // Navigate to catalog
      await tester.tap(find.byKey(Key('catalog_tab')));
      await tester.pumpAndSettle();

      // Search for a product
      await tester.enterText(find.byKey(Key('search_field')), 'laptop');
      await tester.testTextInput.receiveAction(TextInputAction.search);
      await tester.pumpAndSettle();

      // Add first product to cart
      await tester.tap(find.byKey(Key('add_to_cart_button')).first);
      await tester.pumpAndSettle();

      // Verify success message
      expect(find.text('Added to cart'), findsOneWidget);

      // Verify cart badge updates
      expect(find.text('1'), findsOneWidget);

      // Navigate to cart
      await tester.tap(find.byKey(Key('cart_tab')));
      await tester.pumpAndSettle();

      // Verify item in cart
      expect(find.byType(CartItemWidget), findsOneWidget);
      expect(find.text('laptop'), findsOneWidget);

      // Update quantity
      await tester.tap(find.byKey(Key('quantity_increase')));
      await tester.pumpAndSettle();

      // Verify quantity changed
      expect(find.text('2'), findsOneWidget);

      // Apply a coupon
      await tester.enterText(find.byKey(Key('coupon_field')), 'SAVE10');
      await tester.tap(find.byKey(Key('apply_coupon_button')));
      await tester.pumpAndSettle();

      // Proceed to checkout
      await tester.tap(find.text('Proceed to Checkout'));
      await tester.pumpAndSettle();

      // Fill shipping address
      await tester.enterText(
        find.byKey(Key('shipping_address_field')), 
        '123 Test Street, Test City, TS 12345'
      );
      
      // Select payment method
      await tester.tap(find.byKey(Key('payment_method_card')));
      await tester.pumpAndSettle();

      // Enter payment details
      await tester.enterText(find.byKey(Key('card_number')), '4111111111111111');
      await tester.enterText(find.byKey(Key('card_expiry')), '12/25');
      await tester.enterText(find.byKey(Key('card_cvv')), '123');

      // Place order
      await tester.tap(find.text('Place Order'));
      await tester.pumpAndSettle(Duration(seconds: 5));

      // Verify order confirmation
      expect(find.text('Order Placed Successfully'), findsOneWidget);
      expect(find.textContaining('Order #'), findsOneWidget);

      // Navigate to order history
      await tester.tap(find.byKey(Key('view_orders_button')));
      await tester.pumpAndSettle();

      // Verify order appears in history
      expect(find.byType(OrderCard), findsOneWidget);
      expect(find.text('Pending'), findsOneWidget);
    });

    testWidgets('admin can manage products and promotions', (tester) async {
      app.main();
      await tester.pumpAndSettle();

      // Login as admin
      await tester.tap(find.byKey(Key('admin_login_button')));
      await tester.pumpAndSettle();

      await tester.enterText(find.byKey(Key('email_field')), 'admin@test.com');
      await tester.enterText(find.byKey(Key('password_field')), 'password123');
      
      await tester.tap(find.byKey(Key('login_submit_button')));
      await tester.pumpAndSettle(Duration(seconds: 3));

      // Verify admin dashboard
      expect(find.text('Admin Dashboard'), findsOneWidget);

      // Navigate to product management
      await tester.tap(find.text('Products'));
      await tester.pumpAndSettle();

      // Add new product
      await tester.tap(find.byIcon(Icons.add));
      await tester.pumpAndSettle();

      await tester.enterText(find.byKey(Key('product_name')), 'Test Product');
      await tester.enterText(find.byKey(Key('product_description')), 'Test Description');
      await tester.enterText(find.byKey(Key('b2c_price')), '29.99');
      await tester.enterText(find.byKey(Key('b2b_price')), '24.99');
      await tester.enterText(find.byKey(Key('stock_quantity')), '100');

      await tester.tap(find.text('Save Product'));
      await tester.pumpAndSettle(Duration(seconds: 3));

      // Verify product was created
      expect(find.text('Product created successfully'), findsOneWidget);
      expect(find.text('Test Product'), findsOneWidget);

      // Navigate to promotions
      await tester.tap(find.text('Promotions'));
      await tester.pumpAndSettle();

      // Create new promotion
      await tester.tap(find.byIcon(Icons.add));
      await tester.pumpAndSettle();

      await tester.enterText(find.byKey(Key('promotion_name')), 'Weekend Sale');
      await tester.enterText(find.byKey(Key('promotion_description')), '20% off all items');
      await tester.enterText(find.byKey(Key('discount_value')), '20');

      await tester.tap(find.byKey(Key('promotion_type_percentage')));
      await tester.tap(find.byKey(Key('target_audience_both')));

      await tester.tap(find.text('Create Promotion'));
      await tester.pumpAndSettle(Duration(seconds: 3));

      // Verify promotion was created
      expect(find.text('Promotion created successfully'), findsOneWidget);
      expect(find.text('Weekend Sale'), findsOneWidget);
    });

    testWidgets('loyalty points system works correctly', (tester) async {
      app.main();
      await tester.pumpAndSettle();

      // Login as customer
      await tester.tap(find.byKey(Key('customer_login_button')));
      await tester.pumpAndSettle();

      await tester.enterText(find.byKey(Key('email_field')), 'loyalty@test.com');
      await tester.enterText(find.byKey(Key('password_field')), 'password123');
      
      await tester.tap(find.byKey(Key('login_submit_button')));
      await tester.pumpAndSettle(Duration(seconds: 3));

      // Navigate to loyalty section
      await tester.tap(find.byKey(Key('loyalty_tab')));
      await tester.pumpAndSettle();

      // Check initial points balance
      expect(find.textContaining('points'), findsOneWidget);

      // Make a purchase to earn points
      await tester.tap(find.byKey(Key('catalog_tab')));
      await tester.pumpAndSettle();

      await tester.tap(find.byKey(Key('add_to_cart_button')).first);
      await tester.pumpAndSettle();

      await tester.tap(find.byKey(Key('cart_tab')));
      await tester.pumpAndSettle();

      await tester.tap(find.text('Proceed to Checkout'));
      await tester.pumpAndSettle();

      // Complete the order (simplified)
      await tester.tap(find.text('Place Order'));
      await tester.pumpAndSettle(Duration(seconds: 5));

      // Check that points were awarded
      await tester.tap(find.byKey(Key('loyalty_tab')));
      await tester.pumpAndSettle();

      // Verify points increased
      expect(find.textContaining('Earned'), findsOneWidget);
    });
  });
}

// Performance Tests
// test/performance/performance_test.dart
import 'package:flutter/services.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:integration_test/integration_test.dart';
import 'package:tradesuper_app/main.dart' as app;

void main() {
  final binding = IntegrationTestWidgetsFlutterBinding.ensureInitialized();

  group('Performance Tests', () {
    testWidgets('app startup performance', (tester) async {
      final stopwatch = Stopwatch()..start();
      
      app.main();
      await tester.pumpAndSettle();
      
      stopwatch.stop();
      
      // App should start within 3 seconds
      expect(stopwatch.elapsedMilliseconds, lessThan(3000));
      
      // Record performance metrics
      await binding.reportData ??= {};
      binding.reportData!['startup_time_ms'] = stopwatch.elapsedMilliseconds;
    });

    testWidgets('product list scrolling performance', (tester) async {
      app.main();
      await tester.pumpAndSettle();

      // Login and navigate to catalog
      await tester.tap(find.byKey(Key('customer_login_button')));
      await tester.pumpAndSettle();

      await tester.enterText(find.byKey(Key('email_field')), 'test@example.com');
      await tester.enterText(find.byKey(Key('password_field')), 'password123');
      await tester.tap(find.byKey(Key('login_submit_button')));
      await tester.pumpAndSettle();

      await tester.tap(find.byKey(Key('catalog_tab')));
      await tester.pumpAndSettle();

      // Measure scrolling performance
      final stopwatch = Stopwatch()..start();
      
      // Scroll down the product list
      for (int i = 0; i < 10; i++) {
        await tester.drag(
          find.byType(ListView),
          Offset(0, -300),
        );
        await tester.pump();
      }
      
      stopwatch.stop();
      
      // Scrolling should be smooth (less than 100ms total for 10 scrolls)
      expect(stopwatch.elapsedMilliseconds, lessThan(100));
      
      binding.reportData!['scroll_performance_ms'] = stopwatch.elapsedMilliseconds;
    });

    testWidgets('memory usage during cart operations', (tester) async {
      app.main();
      await tester.pumpAndSettle();

      // Perform multiple cart operations and monitor memory
      final List<int> memorySnapshots = [];
      
      // Login
      await tester.tap(find.byKey(Key('customer_login_button')));
      await tester.pumpAndSettle();
      
      // Add memory snapshot
      final SystemChannels.platform.invokeMethod('System.memoryInfo');
      
      // Add multiple items to cart
      for (int i = 0; i < 20; i++) {
        await tester.tap(find.byKey(Key('add_to_cart_button')).first);
        await tester.pumpAndSettle();
      }
      
      // Memory shouldn't increase significantly during cart operations
      // This is a simplified check - in real scenarios you'd use platform channels
      expect(find.byType(CartItemWidget), findsWidgets);
    });
  });
}
```

---

# Phase 5: Production & Monitoring

## 5.1 Production Deployment Strategy

### **Step 24: Infrastructure as Code**

```yaml
# terraform/main.tf
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.23"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

# VPC and Networking
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"
  
  name = "tradesuper-vpc"
  cidr = "10.0.0.0/16"
  
  azs             = ["${var.aws_region}a", "${var.aws_region}b", "${var.aws_region}c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24", "10.0.103.0/24"]
  
  enable_nat_gateway = true
  enable_vpn_gateway = false
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Environment = var.environment
    Project     = "tradesuper"
  }
}

# EKS Cluster
module "eks" {
  source = "terraform-aws-modules/eks});

// Custom metrics collection
const businessMetrics = {
  trackOrder: (orderData) => {
    apm.setCustomContext({
      order: {
        id: orderData.id,
        type: orderData.orderType,
        amount: orderData.totalAmount,
        commission: orderData.commissionAmount,
        userId: orderData.buyerId,
        adminId: orderData.adminId
      }
    });
    
    // Track business metrics
    apm.recordMetric('orders.created', 1);
    apm.recordMetric('revenue.total', orderData.totalAmount);
    apm.recordMetric('commission.earned', orderData.commissionAmount);
    
    if (orderData.orderType === 'B2C') {
      apm.recordMetric('orders.b2c', 1);
      apm.recordMetric('revenue.b2c', orderData.totalAmount);
    } else {
      apm.recordMetric('orders.b2b', 1);
      apm.recordMetric('revenue.b2b', orderData.totalAmount);
    }
  },

  trackLoyaltyPoints: (userId, pointsEarned) => {
    apm.setCustomContext({
      loyalty: { userId, pointsEarned }
    });
    apm.recordMetric('loyalty.points.earned', pointsEarned);
  },

  trackPromotionUsage: (promotionId, discountAmount) => {
    apm.setCustomContext({
      promotion: { id: promotionId, discount: discountAmount }
    });
    apm.recordMetric('promotions.used', 1);
    apm.recordMetric('promotions.discount', discountAmount);
  },

  trackUserActivity: (userId, action, metadata = {}) => {
    apm.setCustomContext({
      user: { id: userId, action, ...metadata }
    });
    apm.recordMetric(`user.activity.${action}`, 1);
  }
};

module.exports = { apm, businessMetrics };

// Enhanced error tracking (utils/errorTracking.js)
const Sentry = require('@sentry/node');
const { ProfilingIntegration } = require('@sentry/profiling-node');

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
  profilesSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
  integrations: [
    new ProfilingIntegration(),
    new Sentry.Integrations.Http({ tracing: true }),
    new Sentry.Integrations.Express({ app: require('../server') }),
    new Sentry.Integrations.Postgres(),
  ],
  beforeSend(event) {
    // Filter out sensitive data
    if (event.user) {
      delete event.user.email;
      delete event.user.ip_address;
    }
    
    if (event.request?.data) {
      const sensitiveFields = ['password', 'token', 'credit_card', 'ssn'];
      sensitiveFields.forEach(field => {
        if (event.request.data[field]) {
          event.request.data[field] = '[Filtered]';
        }
      });
    }
    
    return event;
  },
  beforeBreadcrumb(breadcrumb) {
    // Filter breadcrumbs
    if (breadcrumb.category === 'http' && breadcrumb.data?.url?.includes('/auth/')) {
      return null; // Don't log auth endpoints
    }
    return breadcrumb;
  }
});

// Business-specific error context
const enhanceErrorContext = (error, context = {}) => {
  Sentry.withScope((scope) => {
    // Add business context
    scope.setContext('business', {
      feature: context.feature || 'unknown',
      userRole: context.userRole,
      orderValue: context.orderValue,
      loyaltyTier: context.loyaltyTier,
      adminId: context.adminId
    });

    // Add technical context
    scope.setContext('technical', {
      requestId: context.requestId,
      sessionId: context.sessionId,
      apiVersion: context.apiVersion || 'v1',
      clientVersion: context.clientVersion
    });

    // Set user context (without PII)
    scope.setUser({
      id: context.userId,
      role: context.userRole,
      tier: context.loyaltyTier
    });

    // Add tags for filtering
    scope.setTag('feature', context.feature || 'unknown');
    scope.setTag('userType', context.userRole || 'unknown');
    scope.setTag('orderType', context.orderType);

    Sentry.captureException(error);
  });
};

module.exports = { Sentry, enhanceErrorContext };

// Comprehensive logging (utils/logger.js)
const winston = require('winston');
const DailyRotateFile = require('winston-daily-rotate-file');

const logFormat = winston.format.combine(
  winston.format.timestamp(),
  winston.format.errors({ stack: true }),
  winston.format.json(),
  winston.format.printf(({ timestamp, level, message, stack, ...meta }) => {
    let log = {
      timestamp,
      level,
      message,
      ...meta
    };
    
    if (stack) {
      log.stack = stack;
    }
    
    return JSON.stringify(log);
  })
);

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: logFormat,
  defaultMeta: { 
    service: 'tradesuper-backend',
    version: process.env.APP_VERSION || '1.0.0'
  },
  transports: [
    // Console transport for development
    new winston.transports.Console({
      format: process.env.NODE_ENV === 'development' 
        ? winston.format.combine(
            winston.format.colorize(),
            winston.format.simple()
          )
        : logFormat
    }),

    // File transports for production
    new DailyRotateFile({
      filename: 'logs/application-%DATE%.log',
      datePattern: 'YYYY-MM-DD',
      maxSize: '20m',
      maxFiles: '14d',
      level: 'info'
    }),

    new DailyRotateFile({
      filename: 'logs/error-%DATE%.log',
      datePattern: 'YYYY-MM-DD',
      maxSize: '20m',
      maxFiles: '30d',
      level: 'error'
    }),

    // Business metrics logs
    new DailyRotateFile({
      filename: 'logs/business-%DATE%.log',
      datePattern: 'YYYY-MM-DD',
      maxSize: '20m',
      maxFiles: '90d',
      level: 'info',
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json()
      )
    })
  ]
});

// Business-specific logging methods
logger.business = {
  orderCreated: (orderData) => {
    logger.info('Order created', {
      event: 'order.created',
      orderId: orderData.id,
      orderNumber: orderData.orderNumber,
      orderType: orderData.orderType,
      totalAmount: orderData.totalAmount,
      commission: orderData.commissionAmount,
      buyerId: orderData.buyerId,
      adminId: orderData.adminId,
      timestamp: new Date().toISOString()
    });
  },

  loyaltyPointsAwarded: (userId, points, orderId) => {
    logger.info('Loyalty points awarded', {
      event: 'loyalty.points.awarded',
      userId,
      points,
      orderId,
      timestamp: new Date().toISOString()
    });
  },

  promotionUsed: (promotionId, userId, discountAmount, orderId) => {
    logger.info('Promotion used', {
      event: 'promotion.used',
      promotionId,
      userId,
      discountAmount,
      orderId,
      timestamp: new Date().toISOString()
    });
  },

  userRegistered: (userId, userRole) => {
    logger.info('User registered', {
      event: 'user.registered',
      userId,
      userRole,
      timestamp: new Date().toISOString()
    });
  },

  stockAlert: (productId, currentStock, minLevel) => {
    logger.warn('Low stock alert', {
      event: 'stock.low',
      productId,
      currentStock,
      minLevel,
      timestamp: new Date().toISOString()
    });
  }
};

module.exports = logger;
```

---

# Phase 4: Testing & Quality Assurance

## 4.1 Comprehensive Testing Strategy

### **Step 22: Backend Testing Implementation**

```javascript
// tests/setup.js
const { sequelize } = require('../config/database');
const redis = require('../config/redis');

// Global test setup
beforeAll(async () => {
  // Set test environment
  process.env.NODE_ENV = 'test';
  process.env.JWT_SECRET = 'test_secret';
  
  // Initialize test database
  await sequelize.sync({ force: true });
  
  // Clear Redis
  await redis.flushall();
});

afterAll(async () => {
  // Cleanup
  await sequelize.close();
  await redis.quit();
});

// tests/helpers/testHelpers.js
const jwt = require('jsonwebtoken');
const { User, Product, Order } = require('../../models');

const testHelpers = {
  // User factories
  createUser: async (overrides = {}) => {
    const userData = {
      email: `test${Date.now()}@example.com`,
      password: 'password123',
      fullName: 'Test User',
      role: 'customer',
      status: 'active',
      ...overrides
    };
    return await User.create(userData);
  },

  createAdmin: async (overrides = {}) => {
    return await testHelpers.createUser({
      role: 'admin',
      companyName: 'Test Company',
      ...overrides
    });
  },

  createSupermarket: async (overrides = {}) => {
    return await testHelpers.createUser({
      role: 'supermarket',
      companyName: 'Test Supermarket',
      ...overrides
    });
  },

  // Auth helpers
  generateToken: (userId, role = 'customer') => {
    return jwt.sign({ userId, role }, process.env.JWT_SECRET, { expiresIn: '1h' });
  },

  getAuthHeaders: (userId, role = 'customer') => {
    const token = testHelpers.generateToken(userId, role);
    return { Authorization: `Bearer ${token}` };
  },

  // Product factories
  createProduct: async (adminId, overrides = {}) => {
    const productData = {
      adminId,
      name: `Test Product ${Date.now()}`,
      description: 'Test product description',
      category: 'Test Category',
      b2bPrice: 10.00,
      b2cPrice: 15.00,
      stockQuantity: 100,
      minStockLevel: 10,
      isActive: true,
      ...overrides
    };
    return await Product.create(productData);
  },

  // Order factories
  createOrder: async (buyerId, adminId, overrides = {}) => {
    const orderData = {
      buyerId,
      adminId,
      orderType: 'B2C',
      subtotal: 100.00,
      totalAmount: 100.00,
      commissionRate: 0.07,
      commissionAmount: 7.00,
      status: 'pending',
      paymentStatus: 'pending',
      ...overrides
    };
    return await Order.create(orderData);
  },

  // Database helpers
  clearDatabase: async () => {
    const models = Object.values(sequelize.models);
    await Promise.all(models.map(model => model.destroy({ where: {}, force: true })));
  },

  // Assertion helpers
  expectValidationError: (response, field) => {
    expect(response.status).toBe(400);
    expect(response.body.success).toBe(false);
    expect(response.body.errors).toBeDefined();
    if (field) {
      expect(response.body.errors.some(error => error.field === field)).toBe(true);
    }
  },

  expectAuthError: (response) => {
    expect([401, 403]).toContain(response.status);
    expect(response.body.success).toBe(false);
  },

  expectSuccessResponse: (response, expectedData = {}) => {
    expect(response.status).toBeLessThan(400);
    expect(response.body.success).toBe(true);
    
    Object.keys(expectedData).forEach(key => {
      expect(response.body).toHaveProperty(key, expectedData[key]);
    });
  }
};

module.exports = testHelpers;

// tests/integration/auth.test.js
const request = require('supertest');
const app = require('../../server');
const { User } = require('../../models');
const testHelpers = require('../helpers/testHelpers');

describe('Authentication Integration Tests', () => {
  beforeEach(async () => {
    await testHelpers.clearDatabase();
  });

  describe('POST /api/auth/register', () => {
    it('should register a new customer successfully', async () => {
      const userData = {
        email: 'customer@test.com',
        password: 'password123',
        fullName: 'Test Customer',
        role: 'customer'
      };

      const response = await request(app)
        .post('/api/auth/register')
        .send(userData);

      testHelpers.expectSuccessResponse(response, {
        message: 'User created successfully'
      });

      expect(response.body.token).toBeDefined();
      expect(response.body.user.email).toBe(userData.email);
      expect(response.body.user.role).toBe(userData.role);
      
      // Verify user was created in database
      const user = await User.findOne({ where: { email: userData.email } });
      expect(user).toBeTruthy();
      expect(user.status).toBe('active');
    });

    it('should register a business user with pending status', async () => {
      const userData = {
        email: 'admin@test.com',
        password: 'password123',
        fullName: 'Test Admin',
        role: 'admin',
        companyName: 'Test Company'
      };

      const response = await request(app)
        .post('/api/auth/register')
        .send(userData);

      testHelpers.expectSuccessResponse(response);
      
      const user = await User.findOne({ where: { email: userData.email } });
      expect(user.status).toBe('pending'); // Business users need approval
    });

    it('should reject duplicate email registration', async () => {
      const userData = {
        email: 'duplicate@test.com',
        password: 'password123',
        fullName: 'Test User'
      };

      // Create first user
      await testHelpers.createUser(userData);

      // Try to create second user with same email
      const response = await request(app)
        .post('/api/auth/register')
        .send(userData);

      expect(response.status).toBe(400);
      expect(response.body.message).toContain('already exists');
    });

    it('should validate required fields', async () => {
      const response = await request(app)
        .post('/api/auth/register')
        .send({});

      testHelpers.expectValidationError(response);
    });

    it('should validate email format', async () => {
      const response = await request(app)
        .post('/api/auth/register')
        .send({
          email: 'invalid-email',
          password: 'password123',
          fullName: 'Test User'
        });

      testHelpers.expectValidationError(response, 'email');
    });

    it('should validate password strength', async () => {
      const response = await request(app)
        .post('/api/auth/register')
        .send({
          email: 'test@example.com',
          password: '123', // Too short
          fullName: 'Test User'
        });

      testHelpers.expectValidationError(response, 'password');
    });
  });

  describe('POST /api/auth/login', () => {
    let user;

    beforeEach(async () => {
      user = await testHelpers.createUser({
        email: 'login@test.com',
        password: 'password123'
      });
    });

    it('should login with valid credentials', async () => {
      const response = await request(app)
        .post('/api/auth/login')
        .send({
          email: 'login@test.com',
          password: 'password123'
        });

      testHelpers.expectSuccessResponse(response, {
        message: 'Login successful'
      });

      expect(response.body.token).toBeDefined();
      expect(response.body.user.id).toBe(user.id);
      
      // Verify last login was updated
      await user.reload();
      expect(user.lastLogin).toBeTruthy();
    });

    it('should reject invalid email', async () => {
      const response = await request(app)
        .post('/api/auth/login')
        .send({
          email: 'wrong@test.com',
          password: 'password123'
        });

      expect(response.status).toBe(401);
      expect(response.body.message).toBe('Invalid credentials');
    });

    it('should reject invalid password', async () => {
      const response = await request(app)
        .post('/api/auth/login')
        .send({
          email: 'login@test.com',
          password: 'wrongpassword'
        });

      expect(response.status).toBe(401);
      expect(response.body.message).toBe('Invalid credentials');
    });

    it('should reject suspended user', async () => {
      await user.update({ status: 'suspended' });

      const response = await request(app)
        .post('/api/auth/login')
        .send({
          email: 'login@test.com',
          password: 'password123'
        });

      expect(response.status).toBe(401);
      expect(response.body.message).toContain('suspended');
    });

    it('should reject pending user', async () => {
      await user.update({ status: 'pending' });

      const response = await request(app)
        .post('/api/auth/login')
        .send({
          email: 'login@test.com',
          password: 'password123'
        });

      expect(response.status).toBe(401);
      expect(response.body.message).toContain('pending approval');
    });
  });
});

// tests/integration/cart.test.js
describe('Cart Integration Tests', () => {
  let customer, admin, product, authHeaders;

  beforeEach(async () => {
    await testHelpers.clearDatabase();
    
    customer = await testHelpers.createUser();
    admin = await testHelpers.createAdmin();
    product = await testHelpers.createProduct(admin.id);
    authHeaders = testHelpers.getAuthHeaders(customer.id);
  });

  describe('POST /api/cart/add', () => {
    it('should add product to cart successfully', async () => {
      const response = await request(app)
        .post('/api/cart/add')
        .set(authHeaders)
        .send({
          productId: product.id,
          quantity: 2
        });

      testHelpers.expectSuccessResponse(response, {
        message: 'Item added to cart successfully'
      });

      expect(response.body.cartItem.productId).toBe(product.id);
      expect(response.body.cartItem.quantity).toBe(2);
      expect(response.body.cartItem.unitPrice).toBe(product.b2cPrice);
    });

    it('should update quantity if product already in cart', async () => {
      // Add product first time
      await request(app)
        .post('/api/cart/add')
        .set(authHeaders)
        .send({
          productId: product.id,
          quantity: 1
        });

      // Add same product again
      const response = await request(app)
        .post('/api/cart/add')
        .set(authHeaders)
        .send({
          productId: product.id,
          quantity: 2
        });

      testHelpers.expectSuccessResponse(response);
      expect(response.body.cartItem.quantity).toBe(3); // 1 + 2
    });

    it('should reject if insufficient stock', async () => {
      await product.update({ stockQuantity: 5 });

      const response = await request(app)
        .post('/api/cart/add')
        .set(authHeaders)
        .send({
          productId: product.id,
          quantity: 10 // More than available
        });

      expect(response.status).toBe(400);
      expect(response.body.message).toContain('Insufficient stock');
    });

    it('should use B2B pricing for supermarket users', async () => {
      const supermarket = await testHelpers.createSupermarket();
      const supermarketHeaders = testHelpers.getAuthHeaders(supermarket.id, 'supermarket');

      const response = await request(app)
        .post('/api/cart/add')
        .set(supermarketHeaders)
        .send({
          productId: product.id,
          quantity: 1
        });

      testHelpers.expectSuccessResponse(response);
      expect(response.body.cartItem.unitPrice).toBe(product.b2bPrice);
      expect(response.body.cartItem.priceType).toBe('B2B');
    });

    it('should require authentication', async () => {
      const response = await request(app)
        .post('/api/cart/add')
        .send({
          productId: product.id,
          quantity: 1
        });

      testHelpers.expectAuthError(response);
    });
  });

  describe('GET /api/cart', () => {
    beforeEach(async () => {
      // Add some items to cart
      await request(app)
        .post('/api/cart/add')
        .set(authHeaders)
        .send({ productId: product.id, quantity: 2 });
    });

    it('should return cart with items and totals', async () => {
      const response = await request(app)
        .get('/api/cart')
        .set(authHeaders);

      testHelpers.expectSuccessResponse(response);
      
      const cart = response.body.cart;
      expect(cart.items).toHaveLength(1);
      expect(cart.itemCount).toBe(1);
      expect(cart.subtotal).toBe(product.b2cPrice * 2);
      expect(cart.total).toBe(cart.subtotal);
    });

    it('should return empty cart for new user', async () => {
      const newUser = await testHelpers.createUser();
      const newUserHeaders = testHelpers.getAuthHeaders(newUser.id);

      const response = await request(app)
        .get('/api/cart')
        .set(newUserHeaders);

      testHelpers.expectSuccessResponse(response);
      expect(response.body.cart.items).toHaveLength(0);
      expect(response.body.cart.subtotal).toBe(0);
    });
  });

  describe('PUT /api/cart/item/:itemId', () => {
    let cartItem;

    beforeEach(async () => {
      const addResponse = await request(app)
        .post('/api/cart/add')
        .set(authHeaders)
        .send({ productId: product.id, quantity: 1 });
      
      cartItem = addResponse.body.cartItem;
    });

    it('should update item quantity', async () => {
      const response = await request(app)
        .put(`/api/cart/item/${cartItem.id}`)
        .set(authHeaders)
        .send({ quantity: 5 });

      testHelpers.expectSuccessResponse(response);
      expect(response.body.cartItem.quantity).toBe(5);
    });

    it('should remove item when quantity is 0', async () => {
      const response = await request(app)
        .put(`/api/cart/item/${cartItem.id}`)
        .set(authHeaders)
        .send({ quantity: 0 });

      testHelpers.expectSuccessResponse(response, {
        message: 'Item removed from cart'
      });
    });

    it('should validate stock availability', async () => {
      await product.update({ stockQuantity: 3 });

      const response = await request(app)
        .put(`/api/cart/item/${cartItem.id}`)
        .set(authHeaders)
        .send({ quantity: 5 });

      expect(response.status).toBe(400);
      expect(response.body.message).toContain('Insufficient stock');
    });
  });
});

// tests/integration/orders.test.js
describe('Order Integration Tests', () => {
  let customer, admin, product, cart, authHeaders;

  beforeEach(async () => {
    await testHelpers.clearDatabase();
    
    customer = await testHelpers.createUser();
    admin = await testHelpers.createAdmin();
    product = await testHelpers.createProduct(admin.id, {
      stockQuantity: 100,
      b2cPrice: 25.00
    });
    authHeaders = testHelpers.getAuthHeaders(customer.id);

    // Add item to cart
    await request(app)
      .post('/api/cart/add')
      .set(authHeaders)
      .send({ productId: product.id, quantity: 2 });
  });

  describe('POST /api/orders', () => {
    const orderData = {
      shippingAddress: {
        street: '123 Test St',
        city: 'Test City',
        state: 'TS',
        zipCode: '12345',
        country: 'US'
      },
      paymentMethod: 'credit_card',
      notes: 'Test order notes'
    };

    it('should create order successfully', async () => {
      const response = await request(app)
        .post('/api/orders')
        .set(authHeaders)
        .send(orderData);

      testHelpers.expectSuccessResponse(response, {
        message: 'Order created successfully'
      });

      const order = response.body.order;
      expect(order.orderNumber).toBeDefined();
      expect(order.totalAmount).toBe(50.00); // 2 * 25.00
      expect(order.status).toBe('pending');
      
      // Verify stock was updated
      await product.reload();
      expect(product.stockQuantity).toBe(98); // 100 - 2
    });

    it('should calculate commission correctly', async () => {
      const response = await request(app)
        .post('/api/orders')
        .set(authHeaders)
        .send(orderData);

      const order = response.body.order;
      expect(order.commissionRate).toBe(0.07); // B2C rate
      expect(order.commissionAmount).toBe(3.50); // 7% of 50.00
    });

    it('should clear cart after successful order', async () => {
      await request(app)
        .post('/api/orders')
        .set(authHeaders)
        .send(orderData);

      const cartResponse = await request(app)
        .get('/api/cart')
        .set(authHeaders);

      expect(cartResponse.body.cart.items).toHaveLength(0);
    });

    it('should handle insufficient stock', async () => {
      await product.update({ stockQuantity: 1 });

      const response = await request(app)
        .post('/api/orders')
        .set(authHeaders)
        .send(orderData);

      expect(response.status).toBe(400);
      expect(response.body.message).toContain('Insufficient stock');
    });

    it('should apply loyalty points discount', async () => {
      // Create loyalty account with points
      const loyaltyAccount = await LoyaltyPoints.create({
        userId: customer.id,
        currentBalance: 1000,
        pointsEarned: 1000
      });

      const response = await request(app)
        .post('/api/orders')
        .set(authHeaders)
        .send({
          ...orderData,
          loyaltyPointsUsed: 500 // $5.00 discount
        });

      testHelpers.expectSuccessResponse(response);
      
      const order = response.body.order;
      expect(order.loyaltyPointsUsed).toBe(500);
      expect(order.loyaltyDiscount).toBe(5.00);
      expect(order.totalAmount).toBe(45.00); // 50.00 - 5.00
    });

    it('should require authentication', async () => {
      const response = await request(app)
        .post('/api/orders')
        .send(orderData);

      testHelpers.expectAuthError(response);
    });

    it('should reject empty cart', async () => {
      // Clear cart first
      await request(app)
        .delete('/api/cart/clear')
        .set(authHeaders);

      const response = await request(app)
        .post('/api/orders')
        .set(authHeaders)
        .send(orderData);

      expect(response.status).toBe(400);
      expect(response.body.message).toBe('Cart is empty');
    });
  });
});
```

### **Step 23: Flutter Testing Implementation**

```dart
// test/helpers/test_helpers.dart
import 'package:flutter/material.dart';
import 'package:flutter_test/flutter_test.dart';
import 'package:provider/provider.dart';
import 'package:mockito/mockito.dart';
import 'package:tradesuper_app/data/models/models.dart';
import 'package:tradesuper_app/presentation/providers/providers.dart';

class TestHelpers {
  // Mock data factories
  static User createMockUser({
    int id = 1,
    String email = 'test@example.com',
    String fullName = 'Test User',
    UserRole role = UserRole.customer,
    UserStatus status = UserStatus.active,
  }) {
    return User(
      id: id,
      email: email,
      fullName: fullName,
      role: role,
      status: status,
      createdAt: DateTime.now(),
      updatedAt: DateTime.now(),
    );
  }

  static Product createMockProduct({
    int id = 1,
    int adminId = 1,
    String name = 'Test Product',
    double b2bPrice = 10.0,
    double b2cPrice = 15.0,
    int stockQuantity = 100,
    bool isActive = true,
  }) {
    return Product(
      id: id,
      adminId: adminId,
      name: name,
      b2bPrice: b2bPrice,
      b2cPrice: b2cPrice,
      stockQuantity: stockQuantity,
      minStockLevel: 10,
      maxStockLevel: 1000,
      isActive: isActive,
      createdAt:              name: tradesuper-secrets
              key: DB_USER
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: tradesuper-secrets
              key: DB_PASS
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
  volumeClaimTemplates:
  - metadata:
      name: postgres-storage
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 10Gi

---
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  namespace: tradesuper
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432

---
# k8s/redis.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: tradesuper
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        volumeMounts:
        - name: redis-storage
          mountPath: /data
      volumes:
      - name: redis-storage
        persistentVolumeClaim:
          claimName: redis-pvc

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-pvc
  namespace: tradesuper
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi

---
apiVersion: v1
kind: Service
metadata:
  name: redis-service
  namespace: tradesuper
spec:
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379

---
# k8s/backend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tradesuper-backend
  namespace: tradesuper
  labels:
    app: tradesuper-backend
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: tradesuper-backend
  template:
    metadata:
      labels:
        app: tradesuper-backend
    spec:
      initContainers:
      - name: wait-for-postgres
        image: busybox:1.28
        command: ['sh', '-c']
        args:
        - |
          until nc -z postgres-service 5432; do
            echo "Waiting for postgres..."
            sleep 2
          done
          echo "PostgreSQL is ready!"
      containers:
      - name: backend
        image: tradesuper/backend:latest
        ports:
        - containerPort: 3000
        env:
        - name: NODE_ENV
          valueFrom:
            configMapKeyRef:
              name: tradesuper-config
              key: NODE_ENV
        - name: DB_HOST
          valueFrom:
            secretKeyRef:
              name: tradesuper-secrets
              key: DB_HOST
        - name: DB_NAME
          valueFrom:
            secretKeyRef:
              name: tradesuper-secrets
              key: DB_NAME
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: tradesuper-secrets
              key: DB_USER
        - name: DB_PASS
          valueFrom:
            secretKeyRef:
              name: tradesuper-secrets
              key: DB_PASS
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: tradesuper-secrets
              key: JWT_SECRET
        - name: AWS_ACCESS_KEY_ID
          valueFrom:
            secretKeyRef:
              name: tradesuper-secrets
              key: AWS_ACCESS_KEY_ID
        - name: AWS_SECRET_ACCESS_KEY
          valueFrom:
            secretKeyRef:
              name: tradesuper-secrets
              key: AWS_SECRET_ACCESS_KEY
        - name: B2C_COMMISSION_RATE
          valueFrom:
            configMapKeyRef:
              name: tradesuper-config
              key: B2C_COMMISSION_RATE
        - name: B2B_COMMISSION_RATE
          valueFrom:
            configMapKeyRef:
              name: tradesuper-config
              key: B2B_COMMISSION_RATE
        - name: REDIS_URL
          value: "redis://redis-service:6379"
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          successThreshold: 1
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3
        startupProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 30

---
apiVersion: v1
kind: Service
metadata:
  name: tradesuper-backend-service
  namespace: tradesuper
spec:
  selector:
    app: tradesuper-backend
  ports:
  - name: http
    port: 80
    targetPort: 3000
    protocol: TCP
  type: ClusterIP

---
# k8s/frontend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: tradesuper-frontend
  namespace: tradesuper
  labels:
    app: tradesuper-frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: tradesuper-frontend
  template:
    metadata:
      labels:
        app: tradesuper-frontend
    spec:
      containers:
      - name: frontend
        image: tradesuper/frontend:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10

---
apiVersion: v1
kind: Service
metadata:
  name: tradesuper-frontend-service
  namespace: tradesuper
spec:
  selector:
    app: tradesuper-frontend
  ports:
  - name: http
    port: 80
    targetPort: 80
  type: ClusterIP

---
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tradesuper-ingress
  namespace: tradesuper
  annotations:
    kubernetes.io/ingress.class: "nginx"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/rate-limit: "1000"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
spec:
  tls:
  - hosts:
    - api.tradesuper.com
    - app.tradesuper.com
    secretName: tradesuper-tls
  rules:
  - host: api.tradesuper.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: tradesuper-backend-service
            port:
              number: 80
  - host: app.tradesuper.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: tradesuper-frontend-service
            port:
              number: 80

---
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: tradesuper-backend-hpa
  namespace: tradesuper
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: tradesuper-backend
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 50
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15

---
# k8s/cronjobs.yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: expire-loyalty-points
  namespace: tradesuper
spec:
  schedule: "0 2 * * *" # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: expire-points
            image: tradesuper/backend:latest
            command: ["node"]
            args: ["scripts/expirePoints.js"]
            env:
            - name: NODE_ENV
              value: "production"
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: tradesuper-secrets
                  key: DB_HOST
            # ... other env vars
          restartPolicy: OnFailure

---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: activate-promotions
  namespace: tradesuper
spec:
  schedule: "0 * * * *" # Every hour
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: activate-promotions
            image: tradesuper/backend:latest
            command: ["node"]
            args: ["scripts/activatePromotions.js"]
            env:
            - name: NODE_ENV
              value: "production"
            # ... env vars
          restartPolicy: OnFailure

---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: generate-analytics
  namespace: tradesuper
spec:
  schedule: "0 1 * * *" # Daily at 1 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: generate-analytics
            image: tradesuper/backend:latest
            command: ["node"]
            args: ["scripts/generateDailyAnalytics.js"]
            env:
            - name: NODE_ENV
              value: "production"
            # ... env vars
          restartPolicy: OnFailure
```

## 3.2 CI/CD Pipeline Implementation

### **Step 19: GitHub Actions Workflows**

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME_BACKEND: tradesuper/backend
  IMAGE_NAME_FRONTEND: tradesuper/frontend

jobs:
  test-backend:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: tradesuper_test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
      
      redis:
        image: redis:7-alpine
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 6379:6379

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '18'
        cache: 'npm'
        cache-dependency-path: backend/package-lock.json

    - name: Install dependencies
      working-directory: backend
      run: npm ci

    - name: Run linting
      working-directory: backend
      run: npm run lint

    - name: Run security audit
      working-directory: backend
      run: npm audit --audit-level=moderate

    - name: Run unit tests
      working-directory: backend
      run: npm test
      env:
        NODE_ENV: test
        DB_HOST: localhost
        DB_PORT: 5432
        DB_NAME: tradesuper_test
        DB_USER: postgres
        DB_PASS: postgres
        REDIS_URL: redis://localhost:6379
        JWT_SECRET: test_secret

    - name: Run integration tests
      working-directory: backend
      run: npm run test:integration
      env:
        NODE_ENV: test
        DB_HOST: localhost
        DB_PORT: 5432
        DB_NAME: tradesuper_test
        DB_USER: postgres
        DB_PASS: postgres
        REDIS_URL: redis://localhost:6379
        JWT_SECRET: test_secret

    - name: Generate coverage report
      working-directory: backend
      run: npm run test:coverage

    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: backend/coverage/lcov.info
        flags: backend
        name: backend-coverage

  test-frontend:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup Flutter
      uses: subosito/flutter-action@v2
      with:
        flutter-version: '3.16.0'
        channel: 'stable'

    - name: Get dependencies
      working-directory: frontend
      run: flutter pub get

    - name: Analyze code
      working-directory: frontend
      run: flutter analyze

    - name: Run tests
      working-directory: frontend
      run: flutter test --coverage

    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: frontend/coverage/lcov.info
        flags: frontend
        name: frontend-coverage

  security-scan:
    runs-on: ubuntu-latest
    needs: [test-backend, test-frontend]

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Run Trivy vulnerability scanner
      uses: aquasecurity/trivy-action@master
      with:
        scan-type: 'fs'
        scan-ref: '.'
        format: 'sarif'
        output: 'trivy-results.sarif'

    - name: Upload Trivy scan results to GitHub Security tab
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'

  build-backend:
    runs-on: ubuntu-latest
    needs: [test-backend, security-scan]
    if: github.ref == 'refs/heads/main'

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Log in to Container Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME_BACKEND }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=sha,prefix={{branch}}-
          type=raw,value=latest,enable={{is_default_branch}}

    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: backend
        file: backend/Dockerfile
        target: production
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max
        platforms: linux/amd64,linux/arm64

  build-frontend:
    runs-on: ubuntu-latest
    needs: [test-frontend, security-scan]
    if: github.ref == 'refs/heads/main'

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Set up Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: Log in to Container Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: Extract metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME_FRONTEND }}

    - name: Build and push Docker image
      uses: docker/build-push-action@v5
      with:
        context: frontend
        file: frontend/Dockerfile
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

  deploy-staging:
    runs-on: ubuntu-latest
    needs: [build-backend, build-frontend]
    if: github.ref == 'refs/heads/develop'
    environment: staging

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup kubectl
      uses: azure/setup-kubectl@v3
      with:
        version: 'latest'

    - name: Setup Helm
      uses: azure/setup-helm@v3
      with:
        version: 'latest'

    - name: Configure kubectl
      run: |
        echo "${{ secrets.KUBE_CONFIG_STAGING }}" | base64 -d > kubeconfig
        export KUBECONFIG=kubeconfig

    - name: Deploy to staging
      run: |
        helm upgrade --install tradesuper-staging ./helm/tradesuper \
          --namespace tradesuper-staging \
          --create-namespace \
          --set image.backend.tag=latest \
          --set image.frontend.tag=latest \
          --set environment=staging \
          --set ingress.hosts[0].host=staging-api.tradesuper.com \
          --set ingress.hosts[1].host=staging-app.tradesuper.com

    - name: Run smoke tests
      run: |
        sleep 60  # Wait for deployment
        curl -f https://staging-api.tradesuper.com/health || exit 1
        curl -f https://staging-app.tradesuper.com || exit 1

  deploy-production:
    runs-on: ubuntu-latest
    needs: [build-backend, build-frontend]
    if: github.ref == 'refs/heads/main'
    environment: production

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup kubectl
      uses: azure/setup-kubectl@v3

    - name: Setup Helm
      uses: azure/setup-helm@v3

    - name: Configure kubectl
      run: |
        echo "${{ secrets.KUBE_CONFIG_PROD }}" | base64 -d > kubeconfig
        export KUBECONFIG=kubeconfig

    - name: Database migration
      run: |
        kubectl exec -n tradesuper deployment/tradesuper-backend -- npm run db:migrate

    - name: Deploy to production
      run: |
        helm upgrade --install tradesuper ./helm/tradesuper \
          --namespace tradesuper \
          --set image.backend.tag=latest \
          --set image.frontend.tag=latest \
          --set environment=production \
          --set ingress.hosts[0].host=api.tradesuper.com \
          --set ingress.hosts[1].host=app.tradesuper.com \
          --set monitoring.enabled=true \
          --set autoscaling.enabled=true

    - name: Run health checks
      run: |
        sleep 120  # Wait for deployment
        curl -f https://api.tradesuper.com/health || exit 1
        curl -f https://app.tradesuper.com || exit 1

    - name: Notify deployment success
      uses: 8398a7/action-slack@v3
      with:
        status: success
        channel: '#deployments'
        text: '🚀 TradeSuper deployed to production successfully!'
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

  rollback-on-failure:
    runs-on: ubuntu-latest
    needs: [deploy-production]
    if: failure()
    environment: production

    steps:
    - name: Rollback deployment
      run: |
        helm rollback tradesuper --namespace tradesuper

    - name: Notify rollback
      uses: 8398a7/action-slack@v3
      with:
        status: failure
        channel: '#deployments'
        text: '⚠️ TradeSuper deployment failed and was rolled back!'
      env:
        SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

## 3.3 Monitoring & Observability

### **Step 20: Production Monitoring Setup**

```yaml
# monitoring/prometheus.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
      evaluation_interval: 15s
    
    rule_files:
      - "tradesuper_rules.yml"
    
    alerting:
      alertmanagers:
        - static_configs:
            - targets:
              - alertmanager:9093
    
    scrape_configs:
      - job_name: 'kubernetes-apiservers'
        kubernetes_sd_configs:
        - role: endpoints
        scheme: https
        tls_config:
          ca_file: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        bearer_token_file: /var/run/secrets/kubernetes.io/serviceaccount/token
        relabel_configs:
        - source_labels: [__meta_kubernetes_namespace, __meta_kubernetes_service_name, __meta_kubernetes_endpoint_port_name]
          action: keep
          regex: default;kubernetes;https
      
      - job_name: 'tradesuper-backend'
        kubernetes_sd_configs:
        - role: endpoints
          namespaces:
            names:
            - tradesuper
        relabel_configs:
        - source_labels: [__meta_kubernetes_service_name]
          action: keep
          regex: tradesuper-backend-service
        - source_labels: [__meta_kubernetes_endpoint_port_name]
          action: keep
          regex: metrics
      
      - job_name: 'tradesuper-postgres'
        static_configs:
        - targets: ['postgres-exporter:9187']

  tradesuper_rules.yml: |
    groups:
    - name: tradesuper.rules
      rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value }} errors per second"
      
      - alert: HighResponseTime
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High response time detected"
          description: "95th percentile response time is {{ $value }} seconds"
      
      - alert: DatabaseConnectionsHigh
        expr: pg_stat_database_numbackends / pg_settings_max_connections > 0.8
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Database connections high"
          description: "Database connections are at {{ $value }}% of maximum"
      
      - alert: LoyaltyPointsAnomalous
        expr: increase(loyalty_points_earned_total[1h]) > 10000
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Unusual loyalty points activity"
          description: "{{ $value }} loyalty points earned in the last hour"

---
# monitoring/grafana-dashboard.json (ConfigMap)
apiVersion: v1
kind: ConfigMap
metadata:
  name: tradesuper-dashboard
  namespace: monitoring
data:
  dashboard.json: |
    {
      "dashboard": {
        "id": null,
        "title": "TradeSuper Platform Dashboard",
        "tags": ["tradesuper"],
        "timezone": "browser",
        "panels": [
          {
            "id": 1,
            "title": "Request Rate",
            "type": "graph",
            "targets": [
              {
                "expr": "rate(http_requests_total[5m])",
                "legendFormat": "{{ method }} {{ route }}"
              }
            ],
            "gridPos": {"h": 8, "w": 12, "x": 0, "y": 0}
          },
          {
            "id": 2,
            "title": "Error Rate",
            "type": "graph",
            "targets": [
              {
                "expr": "rate(http_requests_total{status=~\"5..\"}[5m])",
                "legendFormat": "5xx Errors"
              }
            ],
            "gridPos": {"h": 8, "w": 12, "x": 12, "y": 0}
          },
          {
            "id": 3,
            "title": "Response Time",
            "type": "graph",
            "targets": [
              {
                "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))",
                "legendFormat": "95th percentile"
              },
              {
                "expr": "histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m]))",
                "legendFormat": "50th percentile"
              }
            ],
            "gridPos": {"h": 8, "w": 24, "x": 0, "y": 8}
          },
          {
            "id": 4,
            "title": "Business Metrics",
            "type": "stat",
            "targets": [
              {
                "expr": "orders_total",
                "legendFormat": "Total Orders"
              },
              {
                "expr": "revenue_total",
                "legendFormat": "Total Revenue"
              },
              {
                "expr": "active_users_total",
                "legendFormat": "Active Users"
              }
            ],
            "gridPos": {"h": 4, "w": 24, "x": 0, "y": 16}
          }
        ],
        "time": {"from": "now-1h", "to": "now"},
        "refresh": "30s"
      }
    }

---
# monitoring/alertmanager.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoring
data:
  alertmanager.yml: |
    global:
      smtp_smarthost: 'localhost:587'
      smtp_from: 'alerts@tradesuper.com'
      slack_api_url: 'YOUR_SLACK_WEBHOOK_URL'
    
    route:
      group_by: ['alertname']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 1h
      receiver: 'web.hook'
      routes:
      - match:
          severity: critical
        receiver: 'critical-alerts'
      - match:
          severity: warning
        receiver: 'warning-alerts'
    
    receivers:
    - name: 'web.hook'
      webhook_configs:
      - url: 'http://localhost:5001/'
    
    - name: 'critical-alerts'
      slack_configs:
      - channel: '#alerts-critical'
        title: 'Critical Alert - {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
        send_resolved: true
      email_configs:
      - to: 'devops@tradesuper.com'
        subject: 'Critical Alert: {{ .GroupLabels.alertname }}'
        body: |
          {{ range .Alerts }}
          Alert: {{ .Annotations.summary }}
          Description: {{ .Annotations.description }}
          {{ end }}
    
    - name: 'warning-alerts'
      slack_configs:
      - channel: '#alerts-warning'
        title: 'Warning Alert - {{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
        send_resolved: true
```

### **Step 21: Application Performance Monitoring**

```javascript
// Backend APM Integration (utils/apm.js)
const apm = require('elastic-apm-node').start({
  serviceName: 'tradesuper-backend',
  secretToken: process.env.ELASTIC_APM_SECRET_TOKEN,
  serverUrl: process.env.ELASTIC_APM_SERVER_URL,
  environment: process.env.NODE_ENV,
  captureBody: 'errors',
  captureHeaders: true,
  logLevel: 'info',
  metricsInterval: '30s',
  transactionSampleRate: process.env.NODE_ENV === 'production' ? 0.1 : 1.0,
  customMetrics: true!) {
      return 0.0;
    }

    double discount = 0.0;
    switch (promotionType) {
      case PromotionType.percentage:
        discount = (orderAmount * discountValue) / 100;
        break;
      case PromotionType.fixedAmount:
        discount = discountValue;
        break;
      case PromotionType.freeShipping:
        // Handle shipping discount separately
        discount = 0.0;
        break;
      case PromotionType.buyXGetY:
      case PromotionType.bundle:
        // Complex calculation handled elsewhere
        break;
    }

    if (maxDiscountAmount != null && discount > maxDiscountAmount!) {
      discount = maxDiscountAmount!;
    }

    return discount;
  }
}

@JsonEnum()
enum PromotionType {
  @JsonValue('percentage')
  percentage,
  @JsonValue('fixed_amount')
  fixedAmount,
  @JsonValue('buy_x_get_y')
  buyXGetY,
  @JsonValue('free_shipping')
  freeShipping,
  @JsonValue('bundle')
  bundle,
}

// data/models/notification_model.dart
@freezed
class NotificationModel with _$NotificationModel {
  const factory NotificationModel({
    required int id,
    required int userId,
    required String title,
    required String message,
    required NotificationType notificationType,
    required NotificationPriority priority,
    int? relatedId,
    String? relatedType,
    String? actionUrl,
    String? imageUrl,
    required bool isRead,
    required bool isSent,
    bool? sendPush,
    bool? sendEmail,
    bool? sendSms,
    DateTime? scheduledFor,
    DateTime? sentAt,
    DateTime? readAt,
    required DateTime createdAt,
  }) = _NotificationModel;

  factory NotificationModel.fromJson(Map<String, dynamic> json) => 
      _$NotificationModelFromJson(json);

  const NotificationModel._();

  NotificationModel copyWith({
    int? id,
    int? userId,
    String? title,
    String? message,
    NotificationType? notificationType,
    NotificationPriority? priority,
    int? relatedId,
    String? relatedType,
    String? actionUrl,
    String? imageUrl,
    bool? isRead,
    bool? isSent,
    bool? sendPush,
    bool? sendEmail,
    bool? sendSms,
    DateTime? scheduledFor,
    DateTime? sentAt,
    DateTime? readAt,
    DateTime? createdAt,
  }) {
    return NotificationModel(
      id: id ?? this.id,
      userId: userId ?? this.userId,
      title: title ?? this.title,
      message: message ?? this.message,
      notificationType: notificationType ?? this.notificationType,
      priority: priority ?? this.priority,
      relatedId: relatedId ?? this.relatedId,
      relatedType: relatedType ?? this.relatedType,
      actionUrl: actionUrl ?? this.actionUrl,
      imageUrl: imageUrl ?? this.imageUrl,
      isRead: isRead ?? this.isRead,
      isSent: isSent ?? this.isSent,
      sendPush: sendPush ?? this.sendPush,
      sendEmail: sendEmail ?? this.sendEmail,
      sendSms: sendSms ?? this.sendSms,
      scheduledFor: scheduledFor ?? this.scheduledFor,
      sentAt: sentAt ?? this.sentAt,
      readAt: readAt ?? this.readAt,
      createdAt: createdAt ?? this.createdAt,
    );
  }

  Color get priorityColor {
    switch (priority) {
      case NotificationPriority.low:
        return Colors.grey;
      case NotificationPriority.medium:
        return Colors.blue;
      case NotificationPriority.high:
        return Colors.orange;
      case NotificationPriority.urgent:
        return Colors.red;
    }
  }

  IconData get typeIcon {
    switch (notificationType) {
      case NotificationType.orderUpdate:
        return Icons.shopping_bag;
      case NotificationType.promotion:
        return Icons.local_offer;
      case NotificationType.stockAlert:
        return Icons.inventory;
      case NotificationType.system:
        return Icons.settings;
      case NotificationType.loyaltyPoints:
        return Icons.stars;
      case NotificationType.coupon:
        return Icons.card_giftcard;
      case NotificationType.payment:
        return Icons.payment;
      case NotificationType.review:
        return Icons.rate_review;
    }
  }

  String get timeAgo {
    final now = DateTime.now();
    final difference = now.difference(createdAt);

    if (difference.inDays > 0) {
      return '${difference.inDays}d ago';
    } else if (difference.inHours > 0) {
      return '${difference.inHours}h ago';
    } else if (difference.inMinutes > 0) {
      return '${difference.inMinutes}m ago';
    } else {
      return 'Just now';
    }
  }
}

@JsonEnum()
enum NotificationType {
  @JsonValue('order_update')
  orderUpdate,
  @JsonValue('promotion')
  promotion,
  @JsonValue('stock_alert')
  stockAlert,
  @JsonValue('system')
  system,
  @JsonValue('loyalty_points')
  loyaltyPoints,
  @JsonValue('coupon')
  coupon,
  @JsonValue('payment')
  payment,
  @JsonValue('review')
  review,
}

@JsonEnum()
enum NotificationPriority {
  @JsonValue('low')
  low,
  @JsonValue('medium')
  medium,
  @JsonValue('high')
  high,
  @JsonValue('urgent')
  urgent,
}
```

### **Step 16: State Management with Providers**

```dart
// presentation/providers/auth_provider.dart
import 'package:flutter/foundation.dart';
import 'package:injectable/injectable.dart';
import '../../core/services/storage_service.dart';
import '../../data/repositories/auth_repository.dart';
import '../../data/models/user_model.dart';

@injectable
class AuthProvider with ChangeNotifier {
  final AuthRepository _authRepository;
  final StorageService _storageService;

  AuthProvider(this._authRepository, this._storageService);

  User? _currentUser;
  bool _isLoading = false;
  String? _error;
  bool _isInitialized = false;

  User? get currentUser => _currentUser;
  bool get isLoading => _isLoading;
  String? get error => _error;
  bool get isAuthenticated => _currentUser != null;
  bool get isInitialized => _isInitialized;
  
  UserRole? get userRole => _currentUser?.role;
  bool get isOwner => userRole == UserRole.owner;
  bool get isAdmin => userRole == UserRole.admin;
  bool get isSupermarket => userRole == UserRole.supermarket;
  bool get isCustomer => userRole == UserRole.customer;
  bool get isBusinessUser => isAdmin || isSupermarket;

  Future<void> initialize() async {
    if (_isInitialized) return;
    
    _setLoading(true);
    try {
      final token = await _storageService.getAuthToken();
      if (token != null) {
        // Verify token and get user data
        final userResult = await _authRepository.getCurrentUser();
        userResult.fold(
          (failure) => _clearAuth(),
          (user) => _currentUser = user,
        );
      }
    } catch (e) {
      debugPrint('Auth initialization error: $e');
    } finally {
      _isInitialized = true;
      _setLoading(false);
    }
  }

  Future<bool> login({
    required String email,
    required String password,
  }) async {
    _setLoading(true);
    _clearError();

    try {
      final result = await _authRepository.login(
        email: email,
        password: password,
      );

      return result.fold(
        (failure) {
          _setError(failure.message);
          return false;
        },
        (authResponse) async {
          _currentUser = authResponse.user;
          await _storageService.saveAuthToken(authResponse.token);
          await _storageService.saveUserRole(authResponse.user.role.name);
          _clearError();
          return true;
        },
      );
    } catch (e) {
      _setError('An unexpected error occurred');
      return false;
    } finally {
      _setLoading(false);
    }
  }

  Future<bool> register({
    required String email,
    required String password,
    required String fullName,
    String? phone,
    UserRole role = UserRole.customer,
    String? companyName,
    String? taxId,
  }) async {
    _setLoading(true);
    _clearError();

    try {
      final result = await _authRepository.register(
        email: email,
        password: password,
        fullName: fullName,
        phone: phone,
        role: role,
        companyName: companyName,
        taxId: taxId,
      );

      return result.fold(
        (failure) {
          _setError(failure.message);
          return false;
        },
        (authResponse) async {
          _currentUser = authResponse.user;
          await _storageService.saveAuthToken(authResponse.token);
          await _storageService.saveUserRole(authResponse.user.role.name);
          _clearError();
          return true;
        },
      );
    } catch (e) {
      _setError('Registration failed. Please try again.');
      return false;
    } finally {
      _setLoading(false);
    }
  }

  Future<void> logout() async {
    _setLoading(true);
    try {
      await _authRepository.logout();
    } catch (e) {
      debugPrint('Logout error: $e');
    } finally {
      await _clearAuth();
      _setLoading(false);
    }
  }

  Future<bool> forgotPassword(String email) async {
    _setLoading(true);
    _clearError();

    try {
      final result = await _authRepository.forgotPassword(email);
      return result.fold(
        (failure) {
          _setError(failure.message);
          return false;
        },
        (_) {
          _clearError();
          return true;
        },
      );
    } catch (e) {
      _setError('Failed to send reset email');
      return false;
    } finally {
      _setLoading(false);
    }
  }

  Future<bool> resetPassword({
    required String token,
    required String newPassword,
  }) async {
    _setLoading(true);
    _clearError();

    try {
      final result = await _authRepository.resetPassword(
        token: token,
        newPassword: newPassword,
      );
      return result.fold(
        (failure) {
          _setError(failure.message);
          return false;
        },
        (_) {
          _clearError();
          return true;
        },
      );
    } catch (e) {
      _setError('Failed to reset password');
      return false;
    } finally {
      _setLoading(false);
    }
  }

  Future<bool> updateProfile({
    String? fullName,
    String? phone,
    String? companyName,
    Map<String, dynamic>? address,
  }) async {
    if (_currentUser == null) return false;

    _setLoading(true);
    _clearError();

    try {
      final result = await _authRepository.updateProfile(
        fullName: fullName,
        phone: phone,
        companyName: companyName,
        address: address,
      );

      return result.fold(
        (failure) {
          _setError(failure.message);
          return false;
        },
        (updatedUser) {
          _currentUser = updatedUser;
          _clearError();
          return true;
        },
      );
    } catch (e) {
      _setError('Failed to update profile');
      return false;
    } finally {
      _setLoading(false);
    }
  }

  Future<bool> changePassword({
    required String currentPassword,
    required String newPassword,
  }) async {
    _setLoading(true);
    _clearError();

    try {
      final result = await _authRepository.changePassword(
        currentPassword: currentPassword,
        newPassword: newPassword,
      );

      return result.fold(
        (failure) {
          _setError(failure.message);
          return false;
        },
        (_) {
          _clearError();
          return true;
        },
      );
    } catch (e) {
      _setError('Failed to change password');
      return false;
    } finally {
      _setLoading(false);
    }
  }

  Future<void> _clearAuth() async {
    _currentUser = null;
    await _storageService.clearAuthToken();
    await _storageService.clearUserRole();
    notifyListeners();
  }

  void _setLoading(bool loading) {
    _isLoading = loading;
    notifyListeners();
  }

  void _setError(String error) {
    _error = error;
    notifyListeners();
  }

  void _clearError() {
    _error = null;
    notifyListeners();
  }

  // Helper methods for route guards
  bool canAccessOwnerRoutes() => isOwner;
  bool canAccessAdminRoutes() => isAdmin || isOwner;
  bool canAccessBusinessRoutes() => isBusinessUser || isOwner;
  bool canAccessCustomerRoutes() => isCustomer || isOwner;
}

// presentation/providers/cart_provider.dart
@injectable
class CartProvider with ChangeNotifier {
  final CartRepository _cartRepository;
  final ProductRepository _productRepository;
  final AuthProvider _authProvider;

  CartProvider(
    this._cartRepository,
    this._productRepository,
    this._authProvider,
  );

  Cart? _cart;
  bool _isLoading = false;
  String? _error;
  String? _appliedCouponCode;
  int _loyaltyPointsToUse = 0;

  Cart? get cart => _cart;
  bool get isLoading => _isLoading;
  String? get error => _error;
  String? get appliedCouponCode => _appliedCouponCode;
  int get loyaltyPointsToUse => _loyaltyPointsToUse;
  
  int get itemCount => _cart?.itemCount ?? 0;
  double get subtotal => _cart?.subtotal ?? 0.0;
  double get total => _cart?.total ?? 0.0;
  double get totalSavings => _cart?.totalSavings ?? 0.0;
  bool get hasItems => _cart?.isNotEmpty ?? false;
  bool get isEmpty => _cart?.isEmpty ?? true;

  Future<void> loadCart() async {
    if (!_authProvider.isAuthenticated) return;

    _setLoading(true);
    _clearError();

    try {
      final result = await _cartRepository.getCart();
      result.fold(
        (failure) => _setError(failure.message),
        (cart) {
          _cart = cart;
          _clearError();
        },
      );
    } catch (e) {
      _setError('Failed to load cart');
    } finally {
      _setLoading(false);
    }
  }

  Future<bool> addToCart({
    required Product product,
    int quantity = 1,
    String? notes,
  }) async {
    if (!_authProvider.isAuthenticated) {
      _setError('Please login to add items to cart');
      return false;
    }

    _setLoading(true);
    _clearError();

    try {
      final result = await _cartRepository.addToCart(
        productId: product.id,
        quantity: quantity,
        notes: notes,
      );

      return result.fold(
        (failure) {
          _setError(failure.message);
          return false;
        },
        (cartItem) async {
          await loadCart(); // Refresh cart
          return true;
        },
      );
    } catch (e) {
      _setError('Failed to add item to cart');
      return false;
    } finally {
      _setLoading(false);
    }
  }

  Future<bool> updateQuantity({
    required int itemId,
    required int newQuantity,
  }) async {
    if (newQuantity <= 0) {
      return await removeItem(itemId);
    }

    _setLoading(true);
    _clearError();

    try {
      final result = await _cartRepository.updateCartItem(
        itemId: itemId,
        quantity: newQuantity,
      );

      return result.fold(
        (failure) {
          _setError(failure.message);
          return false;
        },
        (_) async {
          await loadCart(); // Refresh cart
          return true;
        },
      );
    } catch (e) {
      _setError('Failed to update quantity');
      return false;
    } finally {
      _setLoading(false);
    }
  }

  Future<bool> removeItem(int itemId) async {
    _setLoading(true);
    _clearError();

    try {
      final result = await _cartRepository.removeCartItem(itemId);
      return result.fold(
        (failure) {
          _setError(failure.message);
          return false;
        },
        (_) async {
          await loadCart(); // Refresh cart
          return true;
        },
      );
    } catch (e) {
      _setError('Failed to remove item');
      return false;
    } finally {
      _setLoading(false);
    }
  }

  Future<bool> clearCart() async {
    _setLoading(true);
    _clearError();

    try {
      final result = await _cartRepository.clearCart();
      return result.fold(
        (failure) {
          _setError(failure.message);
          return false;
        },
        (_) {
          _cart = null;
          _appliedCouponCode = null;
          _loyaltyPointsToUse = 0;
          _clearError();
          return true;
        },
      );
    } catch (e) {
      _setError('Failed to clear cart');
      return false;
    } finally {
      _setLoading(false);
    }
  }

  Future<bool> applyCoupon(String couponCode) async {
    if (_cart == null || _cart!.isEmpty) {
      _setError('Cart is empty');
      return false;
    }

    _setLoading(true);
    _clearError();

    try {
      final cartItems = _cart!.items.map((item) => {
        'productId': item.productId,
        'category': item.product.category,
        'unitPrice': item.unitPrice,
        'quantity': item.quantity,
      }).toList();

      final result = await _cartRepository.validateCoupon(
        code: couponCode,
        orderAmount: _cart!.subtotal,
        cartItems: cartItems,
      );

      return result.fold(
        (failure) {
          _setError(failure.message);
          return false;
        },
        (validation) {
          if (validation.valid) {
            _appliedCouponCode = couponCode;
            await loadCart(); // Refresh cart with discount
            return true;
          } else {
            _setError('Invalid coupon code');
            return false;
          }
        },
      );
    } catch (e) {
      _setError('Failed to apply coupon');
      return false;
    } finally {
      _setLoading(false);
    }
  }

  void removeCoupon() {
    _appliedCouponCode = null;
    loadCart(); // Refresh cart without coupon
  }

  void setLoyaltyPointsToUse(int points) {
    _loyaltyPointsToUse = points;
    notifyListeners();
  }

  Future<CartValidation?> validateCart() async {
    if (_cart == null) return null;

    try {
      final result = await _cartRepository.validateCart();
      return result.fold(
        (failure) {
          _setError(failure.message);
          return null;
        },
        (validation) => validation,
      );
    } catch (e) {
      _setError('Failed to validate cart');
      return null;
    }
  }

  bool isInCart(int productId) {
    return _cart?.hasProduct(productId) ?? false;
  }

  int getQuantityInCart(int productId) {
    final item = _cart?.getItemByProductId(productId);
    return item?.quantity ?? 0;
  }

  CartItem? getCartItem(int productId) {
    return _cart?.getItemByProductId(productId);
  }

  double getItemTotal(int productId) {
    final item = getCartItem(productId);
    return item?.totalPrice ?? 0.0;
  }

  void _setLoading(bool loading) {
    _isLoading = loading;
    notifyListeners();
  }

  void _setError(String error) {
    _error = error;
    notifyListeners();
  }

  void _clearError() {
    _error = null;
    notifyListeners();
  }

  @override
  void dispose() {
    super.dispose();
  }
}

// presentation/providers/product_provider.dart
@injectable
class ProductProvider with ChangeNotifier {
  final ProductRepository _productRepository;

  ProductProvider(this._productRepository);

  List<Product> _products = [];
  List<Product> _featuredProducts = [];
  List<String> _categories = [];
  List<String> _brands = [];
  bool _isLoading = false;
  bool _isLoadingMore = false;
  String? _error;
  
  // Pagination
  int _currentPage = 1;
  bool _hasMore = true;
  
  // Filters
  ProductFilters _filters = const ProductFilters();

  List<Product> get products => _products;
  List<Product> get featuredProducts => _featuredProducts;
  List<String> get categories => _categories;
  List<String> get brands => _brands;
  bool get isLoading => _isLoading;
  bool get isLoadingMore => _isLoadingMore;
  String? get error => _error;
  bool get hasMore => _hasMore;
  ProductFilters get filters => _filters;

  Future<void> loadProducts({bool refresh = false}) async {
    if (refresh) {
      _currentPage = 1;
      _hasMore = true;
      _products.clear();
    }

    if (!_hasMore && !refresh) return;

    _setLoading(refresh);
    _setLoadingMore(!refresh);
    _clearError();

    try {
      final result = await _productRepository.getProducts(
        page: _currentPage,
        limit: 20,
        category: _filters.category,
        brand: _filters.brand,
        search: _filters.search,
        minPrice: _filters.minPrice,
        maxPrice: _filters.maxPrice,
        userType: _filters.userType,
        inStock: _filters.inStock,
        sortBy: _filters.sortBy,
        sortOrder: _filters.sortOrder,
      );

      result.fold(
        (failure) => _setError(failure.message),
        (response) {
          if (refresh) {
            _products = response.products;
          } else {
            _products.addAll(response.products);
          }
          
          _hasMore = response.pagination.currentPage < response.pagination.totalPages;
          if (_hasMore) _currentPage++;
          
          _clearError();
        },
      );
    } catch (e) {
      _setError('Failed to load products');
    } finally {
      _setLoading(false);
      _setLoadingMore(false);
    }
  }

  Future<void> loadFeaturedProducts() async {
    try {
      final result = await _productRepository.getFeaturedProducts();
      result.fold(
        (failure) => debugPrint('Failed to load featured products: ${failure.message}'),
        (products) => _featuredProducts = products,
      );
    } catch (e) {
      debugPrint('Featured products error: $e');
    }
    notifyListeners();
  }

  Future<void> loadCategories() async {
    try {
      final result = await _productRepository.getCategories();
      result.fold(
        (failure) => debugPrint('Failed to load categories: ${failure.message}'),
        (categories) => _categories = categories,
      );
    } catch (e) {
      debugPrint('Categories error: $e');
    }
    notifyListeners();
  }

  Future<void> loadBrands() async {
    try {
      final result = await _productRepository.getBrands();
      result.fold(
        (failure) => debugPrint('Failed to load brands: ${failure.message}'),
        (brands) => _brands = brands,
      );
    } catch (e) {
      debugPrint('Brands error: $e');
    }
    notifyListeners();
  }

  Future<Product?> getProduct(int id, {String userType = 'B2C'}) async {
    try {
      final result = await _productRepository.getProduct(id, userType: userType);
      return result.fold(
        (failure) {
          _setError(failure.message);
          return null;
        },
        (product) => product,
      );
    } catch (e) {
      _setError('Failed to load product');
      return null;
    }
  }

  void updateFilters(ProductFilters newFilters) {
    _filters = newFilters;
    loadProducts(refresh: true);
  }

  void clearFilters() {
    _filters = const ProductFilters();
    loadProducts(refresh: true);
  }

  void search(String query) {
    _filters = _filters.copyWith(search: query);
    loadProducts(refresh: true);
  }

  void filterByCategory(String? category) {
    _filters = _filters.copyWith(category: category);
    loadProducts(refresh: true);
  }

  void filterByBrand(String? brand) {
    _filters = _filters.copyWith(brand: brand);
    loadProducts(refresh: true);
  }

  void filterByPriceRange(double? minPrice, double? maxPrice) {
    _filters = _filters.copyWith(minPrice: minPrice, maxPrice: maxPrice);
    loadProducts(refresh: true);
  }

  void sortBy(String sortBy, String sortOrder) {
    _filters = _filters.copyWith(sortBy: sortBy, sortOrder: sortOrder);
    loadProducts(refresh: true);
  }

  void _setLoading(bool loading) {
    _isLoading = loading;
    notifyListeners();
  }

  void _setLoadingMore(bool loading) {
    _isLoadingMore = loading;
    notifyListeners();
  }

  void _setError(String error) {
    _error = error;
    notifyListeners();
  }

  void _clearError() {
    _error = null;
    notifyListeners();
  }
}

@freezed
class ProductFilters with _$ProductFilters {
  const factory ProductFilters({
    String? category,
    String? brand,
    String? search,
    double? minPrice,
    double? maxPrice,
    @Default('B2C') String userType,
    @Default(true) bool inStock,
    @Default('created_at') String sortBy,
    @Default('DESC') String sortOrder,
  }) = _ProductFilters;
}
```

---

# Phase 3: DevOps & Deployment

## 3.1 Containerization & Orchestration

### **Step 17: Production Docker Setup**

```dockerfile
# Backend Production Dockerfile
FROM node:18-alpine AS base
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

FROM node:18-alpine AS development
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
EXPOSE 3000
CMD ["npm", "run", "dev"]

FROM base AS production
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodeuser -u 1001
COPY --chown=nodeuser:nodejs . .
USER nodeuser
EXPOSE 3000
ENV NODE_ENV=production
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
CMD ["npm", "start"]

# Flutter Web Dockerfile
FROM cirrusci/flutter:stable AS build
WORKDIR /app
COPY pubspec.* ./
RUN flutter pub get
COPY . .
RUN flutter build web --release

FROM nginx:alpine AS production
COPY --from=build /app/build/web /usr/share/nginx/html
COPY nginx.conf /etc/nginx/nginx.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### **Step 18: Kubernetes Deployment Configuration**

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: tradesuper
  labels:
    name: tradesuper

---
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: tradesuper-config
  namespace: tradesuper
data:
  NODE_ENV: "production"
  B2C_COMMISSION_RATE: "0.07"
  B2B_COMMISSION_RATE: "0.03"
  REQUIRE_EMAIL_VERIFICATION: "true"

---
# k8s/secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: tradesuper-secrets
  namespace: tradesuper
type: Opaque
stringData:
  DB_HOST: "postgres-service"
  DB_NAME: "tradesuper"
  DB_USER: "tradesuper_user"
  DB_PASS: "your-secure-password"
  JWT_SECRET: "your-super-secret-jwt-key"
  AWS_ACCESS_KEY_ID: "your-aws-access-key"
  AWS_SECRET_ACCESS_KEY: "your-aws-secret-key"
  AWS_REGION: "us-east-1"
  AWS_S3_BUCKET: "tradesuper-images"
  SENTRY_DSN: "your-sentry-dsn"

---
# k8s/postgres.yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
  namespace: tradesuper
spec:
  serviceName: postgres-service
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_DB
          valueFrom:
            secretKeyRef:
              name: tradesuper-secrets
              key: DB_NAME
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:    } on DioException {
      rethrow;
    }
  }

  Future<Response> put(
    String path, {
    dynamic data,
    Map<String, dynamic>? queryParameters,
    Options? options,
    CancelToken? cancelToken,
  }) async {
    try {
      return await _dio.put(
        path,
        data: data,
        queryParameters: queryParameters,
        options: options,
        cancelToken: cancelToken,
      );
    } on DioException {
      rethrow;
    }
  }

  Future<Response> delete(
    String path, {
    Map<String, dynamic>? queryParameters,
    Options? options,
    CancelToken? cancelToken,
  }) async {
    try {
      return await _dio.delete(
        path,
        queryParameters: queryParameters,
        options: options,
        cancelToken: cancelToken,
      );
    } on DioException {
      rethrow;
    }
  }
}

// core/services/api_service.dart
import 'package:injectable/injectable.dart';
import '../network/dio_client.dart';
import '../../data/models/models.dart';

@lazySingleton
class ApiService {
  final DioClient _dioClient;

  ApiService(this._dioClient);

  // Authentication endpoints
  Future<AuthResponse> login({
    required String email,
    required String password,
  }) async {
    final response = await _dioClient.post('/auth/login', data: {
      'email': email,
      'password': password,
    });
    return AuthResponse.fromJson(response.data);
  }

  Future<AuthResponse> register({
    required Map<String, dynamic> userData,
  }) async {
    final response = await _dioClient.post('/auth/register', data: userData);
    return AuthResponse.fromJson(response.data);
  }

  Future<BaseResponse> logout() async {
    final response = await _dioClient.post('/auth/logout');
    return BaseResponse.fromJson(response.data);
  }

  Future<AuthResponse> refreshToken() async {
    final response = await _dioClient.post('/auth/refresh');
    return AuthResponse.fromJson(response.data);
  }

  // Product endpoints
  Future<ProductListResponse> getProducts({
    int page = 1,
    int limit = 20,
    String? category,
    String? search,
    String? sortBy,
    String? sortOrder,
    double? minPrice,
    double? maxPrice,
    String userType = 'B2C',
    bool inStock = true,
  }) async {
    final response = await _dioClient.get('/products', queryParameters: {
      'page': page,
      'limit': limit,
      if (category != null) 'category': category,
      if (search != null) 'search': search,
      if (sortBy != null) 'sortBy': sortBy,
      if (sortOrder != null) 'sortOrder': sortOrder,
      if (minPrice != null) 'minPrice': minPrice,
      if (maxPrice != null) 'maxPrice': maxPrice,
      'userType': userType,
      'inStock': inStock,
    });
    return ProductListResponse.fromJson(response.data);
  }

  Future<ProductResponse> getProduct({
    required int id,
    String userType = 'B2C',
  }) async {
    final response = await _dioClient.get('/products/$id', queryParameters: {
      'userType': userType,
    });
    return ProductResponse.fromJson(response.data);
  }

  // Cart endpoints
  Future<CartResponse> getCart() async {
    final response = await _dioClient.get('/cart');
    return CartResponse.fromJson(response.data);
  }

  Future<CartItemResponse> addToCart({
    required int productId,
    required int quantity,
    String? notes,
  }) async {
    final response = await _dioClient.post('/cart/add', data: {
      'productId': productId,
      'quantity': quantity,
      if (notes != null) 'notes': notes,
    });
    return CartItemResponse.fromJson(response.data);
  }

  Future<CartItemResponse> updateCartItem({
    required int itemId,
    required int quantity,
    String? notes,
  }) async {
    final response = await _dioClient.put('/cart/item/$itemId', data: {
      'quantity': quantity,
      if (notes != null) 'notes': notes,
    });
    return CartItemResponse.fromJson(response.data);
  }

  Future<BaseResponse> removeCartItem(int itemId) async {
    final response = await _dioClient.delete('/cart/item/$itemId');
    return BaseResponse.fromJson(response.data);
  }

  Future<BaseResponse> clearCart() async {
    final response = await _dioClient.delete('/cart/clear');
    return BaseResponse.fromJson(response.data);
  }

  Future<CartValidationResponse> validateCart() async {
    final response = await _dioClient.get('/cart/validate');
    return CartValidationResponse.fromJson(response.data);
  }

  // Order endpoints
  Future<OrderResponse> createOrder({
    required Map<String, dynamic> orderData,
  }) async {
    final response = await _dioClient.post('/orders', data: orderData);
    return OrderResponse.fromJson(response.data);
  }

  Future<OrderListResponse> getOrders({
    int page = 1,
    int limit = 20,
    String? status,
    String? orderType,
    String? startDate,
    String? endDate,
  }) async {
    final response = await _dioClient.get('/orders', queryParameters: {
      'page': page,
      'limit': limit,
      if (status != null) 'status': status,
      if (orderType != null) 'orderType': orderType,
      if (startDate != null) 'startDate': startDate,
      if (endDate != null) 'endDate': endDate,
    });
    return OrderListResponse.fromJson(response.data);
  }

  Future<OrderResponse> getOrder(int id) async {
    final response = await _dioClient.get('/orders/$id');
    return OrderResponse.fromJson(response.data);
  }

  Future<OrderResponse> updateOrderStatus({
    required int id,
    required String status,
    String? trackingNumber,
    String? notes,
  }) async {
    final response = await _dioClient.put('/orders/$id/status', data: {
      'status': status,
      if (trackingNumber != null) 'trackingNumber': trackingNumber,
      if (notes != null) 'internalNotes': notes,
    });
    return OrderResponse.fromJson(response.data);
  }

  Future<OrderResponse> cancelOrder({
    required int id,
    String? reason,
  }) async {
    final response = await _dioClient.put('/orders/$id/cancel', data: {
      if (reason != null) 'reason': reason,
    });
    return OrderResponse.fromJson(response.data);
  }

  // Promotion endpoints
  Future<PromotionListResponse> getActivePromotions({
    String? userType,
    int? productId,
    String? category,
    double? orderAmount,
    bool featured = false,
  }) async {
    final response = await _dioClient.get('/promotions/active', queryParameters: {
      if (userType != null) 'userType': userType,
      if (productId != null) 'productId': productId,
      if (category != null) 'category': category,
      if (orderAmount != null) 'orderAmount': orderAmount,
      'featured': featured,
    });
    return PromotionListResponse.fromJson(response.data);
  }

  Future<PromotionListResponse> getFeaturedPromotions() async {
    final response = await _dioClient.get('/promotions/featured');
    return PromotionListResponse.fromJson(response.data);
  }

  // Coupon endpoints
  Future<CouponValidationResponse> validateCoupon({
    required String code,
    required double orderAmount,
    List<Map<String, dynamic>> cartItems = const [],
  }) async {
    final response = await _dioClient.post('/coupons/validate', data: {
      'code': code,
      'orderAmount': orderAmount,
      'cartItems': cartItems,
    });
    return CouponValidationResponse.fromJson(response.data);
  }

  Future<CouponListResponse> getUserCoupons({
    double? orderAmount,
  }) async {
    final response = await _dioClient.get('/coupons/user', queryParameters: {
      if (orderAmount != null) 'orderAmount': orderAmount,
    });
    return CouponListResponse.fromJson(response.data);
  }

  // Loyalty endpoints
  Future<LoyaltyAccountResponse> getLoyaltyAccount() async {
    final response = await _dioClient.get('/loyalty/points');
    return LoyaltyAccountResponse.fromJson(response.data);
  }

  Future<LoyaltyTransactionListResponse> getLoyaltyTransactions({
    int page = 1,
    int limit = 20,
    String? type,
    String? startDate,
    String? endDate,
  }) async {
    final response = await _dioClient.get('/loyalty/transactions', queryParameters: {
      'page': page,
      'limit': limit,
      if (type != null) 'type': type,
      if (startDate != null) 'startDate': startDate,
      if (endDate != null) 'endDate': endDate,
    });
    return LoyaltyTransactionListResponse.fromJson(response.data);
  }

  Future<LoyaltyRedemptionResponse> redeemPoints({
    required int pointsToRedeem,
  }) async {
    final response = await _dioClient.post('/loyalty/redeem', data: {
      'pointsToRedeem': pointsToRedeem,
    });
    return LoyaltyRedemptionResponse.fromJson(response.data);
  }

  // Notification endpoints
  Future<NotificationListResponse> getNotifications({
    int page = 1,
    int limit = 20,
    bool unreadOnly = false,
  }) async {
    final response = await _dioClient.get('/notifications', queryParameters: {
      'page': page,
      'limit': limit,
      'unreadOnly': unreadOnly,
    });
    return NotificationListResponse.fromJson(response.data);
  }

  Future<BaseResponse> markNotificationAsRead(int notificationId) async {
    final response = await _dioClient.put('/notifications/$notificationId/read');
    return BaseResponse.fromJson(response.data);
  }

  Future<BaseResponse> markAllNotificationsAsRead() async {
    final response = await _dioClient.put('/notifications/mark-all-read');
    return BaseResponse.fromJson(response.data);
  }

  Future<UnreadCountResponse> getUnreadNotificationCount() async {
    final response = await _dioClient.get('/notifications/unread-count');
    return UnreadCountResponse.fromJson(response.data);
  }

  // Wishlist endpoints
  Future<WishlistResponse> getWishlist({
    int page = 1,
    int limit = 20,
  }) async {
    final response = await _dioClient.get('/wishlist', queryParameters: {
      'page': page,
      'limit': limit,
    });
    return WishlistResponse.fromJson(response.data);
  }

  Future<BaseResponse> addToWishlist({
    required int productId,
    String? notes,
  }) async {
    final response = await _dioClient.post('/wishlist/add', data: {
      'productId': productId,
      if (notes != null) 'notes': notes,
    });
    return BaseResponse.fromJson(response.data);
  }

  Future<BaseResponse> removeFromWishlist(int productId) async {
    final response = await _dioClient.delete('/wishlist/$productId');
    return BaseResponse.fromJson(response.data);
  }

  // Analytics endpoints
  Future<AnalyticsResponse> getAnalytics({
    String? period,
    String? startDate,
    String? endDate,
  }) async {
    final response = await _dioClient.get('/analytics', queryParameters: {
      if (period != null) 'period': period,
      if (startDate != null) 'startDate': startDate,
      if (endDate != null) 'endDate': endDate,
    });
    return AnalyticsResponse.fromJson(response.data);
  }
}
```

### **Step 15: Enhanced Data Models**

```dart
// data/models/user_model.dart
import 'package:freezed_annotation/freezed_annotation.dart';

part 'user_model.freezed.dart';
part 'user_model.g.dart';

@freezed
class User with _$User {
  const factory User({
    required int id,
    required String email,
    required String fullName,
    String? phone,
    required UserRole role,
    required UserStatus status,
    bool? emailVerified,
    bool? phoneVerified,
    DateTime? lastLogin,
    String? profileImageUrl,
    String? companyName,
    String? taxId,
    Map<String, dynamic>? address,
    required DateTime createdAt,
    required DateTime updatedAt,
  }) = _User;

  factory User.fromJson(Map<String, dynamic> json) => _$UserFromJson(json);
}

@JsonEnum()
enum UserRole {
  @JsonValue('owner')
  owner,
  @JsonValue('admin')
  admin,
  @JsonValue('supermarket')
  supermarket,
  @JsonValue('customer')
  customer,
}

@JsonEnum()
enum UserStatus {
  @JsonValue('active')
  active,
  @JsonValue('suspended')
  suspended,
  @JsonValue('pending')
  pending,
}

// data/models/product_model.dart
@freezed
class Product with _$Product {
  const factory Product({
    required int id,
    required int adminId,
    required String name,
    String? description,
    String? category,
    String? brand,
    String? sku,
    String? barcode,
    required double b2bPrice,
    required double b2cPrice,
    double? costPrice,
    required int stockQuantity,
    required int minStockLevel,
    required int maxStockLevel,
    String? unit,
    double? weight,
    Map<String, dynamic>? dimensions,
    List<String>? imageUrls,
    String? featuredImageUrl,
    List<String>? tags,
    bool? isFeatured,
    bool? isActive,
    String? metaTitle,
    String? metaDescription,
    required DateTime createdAt,
    required DateTime updatedAt,
    // Computed fields
    double? currentPrice,
    bool? isLowStock,
    bool? isOutOfStock,
    double? margin,
    User? admin,
  }) = _Product;

  factory Product.fromJson(Map<String, dynamic> json) => _$ProductFromJson(json);

  const Product._();

  double getPriceForUserType(String userType) {
    return userType == 'B2B' ? b2bPrice : b2cPrice;
  }

  bool get hasMultipleImages => imageUrls != null && imageUrls!.length > 1;
  
  String get displayPrice {
    if (currentPrice != null) {
      return '\${currentPrice!.toStringAsFixed(2)}';
    }
    return '\${b2cPrice.toStringAsFixed(2)}';
  }

  String get stockStatus {
    if (stockQuantity <= 0) return 'Out of Stock';
    if (stockQuantity <= minStockLevel) return 'Low Stock';
    return 'In Stock';
  }

  Color get stockStatusColor {
    if (stockQuantity <= 0) return Colors.red;
    if (stockQuantity <= minStockLevel) return Colors.orange;
    return Colors.green;
  }
}

// data/models/cart_model.dart
@freezed
class Cart with _$Cart {
  const factory Cart({
    required int id,
    required int userId,
    required List<CartItem> items,
    required double subtotal,
    double? discountAmount,
    double? couponDiscount,
    double? loyaltyDiscount,
    double? shippingCost,
    double? taxAmount,
    required double total,
    required int itemCount,
    List<AppliedPromotion>? appliedPromotions,
    String? couponCode,
    int? loyaltyPointsUsed,
    required DateTime updatedAt,
  }) = _Cart;

  factory Cart.fromJson(Map<String, dynamic> json) => _$CartFromJson(json);

  const Cart._();

  bool get isEmpty => items.isEmpty;
  bool get isNotEmpty => items.isNotEmpty;
  
  double get totalSavings {
    return (discountAmount ?? 0) + 
           (couponDiscount ?? 0) + 
           (loyaltyDiscount ?? 0);
  }

  int getTotalQuantity() {
    return items.fold(0, (sum, item) => sum + item.quantity);
  }

  bool hasProduct(int productId) {
    return items.any((item) => item.productId == productId);
  }

  CartItem? getItemByProductId(int productId) {
    try {
      return items.firstWhere((item) => item.productId == productId);
    } catch (e) {
      return null;
    }
  }
}

@freezed
class CartItem with _$CartItem {
  const factory CartItem({
    required int id,
    required int cartId,
    required int productId,
    required Product product,
    required int quantity,
    required String priceType,
    required double unitPrice,
    String? notes,
    required DateTime createdAt,
    required DateTime updatedAt,
  }) = _CartItem;

  factory CartItem.fromJson(Map<String, dynamic> json) => _$CartItemFromJson(json);

  const CartItem._();

  double get totalPrice => unitPrice * quantity;
  
  bool get isB2B => priceType == 'B2B';
  bool get isB2C => priceType == 'B2C';
}

@freezed
class AppliedPromotion with _$AppliedPromotion {
  const factory AppliedPromotion({
    required int id,
    required String name,
    required double discountAmount,
    String? description,
  }) = _AppliedPromotion;

  factory AppliedPromotion.fromJson(Map<String, dynamic> json) => 
      _$AppliedPromotionFromJson(json);
}

// data/models/order_model.dart
@freezed
class Order with _$Order {
  const factory Order({
    required int id,
    required String orderNumber,
    required int buyerId,
    required int adminId,
    required OrderType orderType,
    required double subtotal,
    double? discountAmount,
    double? couponDiscount,
    int? loyaltyPointsUsed,
    double? loyaltyDiscount,
    double? shippingCost,
    double? taxAmount,
    required double totalAmount,
    required double commissionRate,
    required double commissionAmount,
    required OrderStatus status,
    required PaymentStatus paymentStatus,
    Map<String, dynamic>? shippingAddress,
    Map<String, dynamic>? billingAddress,
    String? paymentMethod,
    String? paymentReference,
    int? promotionId,
    int? couponId,
    DateTime? deliveryDate,
    String? deliveryTimeSlot,
    String? trackingNumber,
    String? notes,
    String? internalNotes,
    String? cancelledReason,
    double? refundAmount,
    required DateTime createdAt,
    required DateTime updatedAt,
    // Relations
    User? buyer,
    User? admin,
    List<OrderItem>? orderItems,
    Promotion? promotion,
    Coupon? coupon,
    // Computed fields
    bool? canBeCancelled,
    bool? canBeRefunded,
    double? totalSavings,
  }) = _Order;

  factory Order.fromJson(Map<String, dynamic> json) => _$OrderFromJson(json);

  const Order._();

  String get statusDisplayName {
    switch (status) {
      case OrderStatus.pending:
        return 'Pending';
      case OrderStatus.confirmed:
        return 'Confirmed';
      case OrderStatus.processing:
        return 'Processing';
      case OrderStatus.shipped:
        return 'Shipped';
      case OrderStatus.delivered:
        return 'Delivered';
      case OrderStatus.cancelled:
        return 'Cancelled';
      case OrderStatus.refunded:
        return 'Refunded';
    }
  }

  Color get statusColor {
    switch (status) {
      case OrderStatus.pending:
        return Colors.orange;
      case OrderStatus.confirmed:
      case OrderStatus.processing:
        return Colors.blue;
      case OrderStatus.shipped:
        return Colors.purple;
      case OrderStatus.delivered:
        return Colors.green;
      case OrderStatus.cancelled:
      case OrderStatus.refunded:
        return Colors.red;
    }
  }

  IconData get statusIcon {
    switch (status) {
      case OrderStatus.pending:
        return Icons.schedule;
      case OrderStatus.confirmed:
        return Icons.check_circle_outline;
      case OrderStatus.processing:
        return Icons.settings;
      case OrderStatus.shipped:
        return Icons.local_shipping;
      case OrderStatus.delivered:
        return Icons.check_circle;
      case OrderStatus.cancelled:
        return Icons.cancel;
      case OrderStatus.refunded:
        return Icons.money_off;
    }
  }

  bool get isCompleted => status == OrderStatus.delivered;
  bool get isCancellable => [OrderStatus.pending, OrderStatus.confirmed].contains(status);
  bool get isRefundable => status == OrderStatus.delivered && paymentStatus == PaymentStatus.paid;
  
  int get totalItems {
    return orderItems?.fold(0, (sum, item) => sum + item.quantity) ?? 0;
  }
}

@JsonEnum()
enum OrderType {
  @JsonValue('B2B')
  b2b,
  @JsonValue('B2C')
  b2c,
}

@JsonEnum()
enum OrderStatus {
  @JsonValue('pending')
  pending,
  @JsonValue('confirmed')
  confirmed,
  @JsonValue('processing')
  processing,
  @JsonValue('shipped')
  shipped,
  @JsonValue('delivered')
  delivered,
  @JsonValue('cancelled')
  cancelled,
  @JsonValue('refunded')
  refunded,
}

@JsonEnum()
enum PaymentStatus {
  @JsonValue('pending')
  pending,
  @JsonValue('paid')
  paid,
  @JsonValue('failed')
  failed,
  @JsonValue('refunded')
  refunded,
  @JsonValue('partial')
  partial,
}

@freezed
class OrderItem with _$OrderItem {
  const factory OrderItem({
    required int id,
    required int orderId,
    required int productId,
    required String productName,
    String? productSku,
    required int quantity,
    required double unitPrice,
    required double totalPrice,
    double? discountAmount,
    required DateTime createdAt,
    Product? product,
  }) = _OrderItem;

  factory OrderItem.fromJson(Map<String, dynamic> json) => _$OrderItemFromJson(json);
}

// data/models/loyalty_model.dart
@freezed
class LoyaltyAccount with _$LoyaltyAccount {
  const factory LoyaltyAccount({
    required int id,
    required int userId,
    required int pointsEarned,
    required int pointsSpent,
    required int pointsExpired,
    required int currentBalance,
    required String tierLevel,
    required int tierProgress,
    required double lifetimeValue,
    required DateTime createdAt,
    required DateTime updatedAt,
    // Computed fields
    double? pointsValue,
    int? pointsToNextTier,
    double? tierMultiplier,
    int? expiringPoints,
  }) = _LoyaltyAccount;

  factory LoyaltyAccount.fromJson(Map<String, dynamic> json) => 
      _$LoyaltyAccountFromJson(json);

  const LoyaltyAccount._();

  Color get tierColor {
    switch (tierLevel.toLowerCase()) {
      case 'bronze':
        return const Color(0xFFCD7F32);
      case 'silver':
        return const Color(0xFFC0C0C0);
      case 'gold':
        return const Color(0xFFFFD700);
      case 'platinum':
        return const Color(0xFFE5E4E2);
      case 'diamond':
        return const Color(0xFFB9F2FF);
      default:
        return Colors.grey;
    }
  }

  IconData get tierIcon {
    switch (tierLevel.toLowerCase()) {
      case 'bronze':
        return Icons.workspace_premium;
      case 'silver':
        return Icons.military_tech;
      case 'gold':
        return Icons.emoji_events;
      case 'platinum':
        return Icons.diamond;
      case 'diamond':
        return Icons.star;
      default:
        return Icons.badge;
    }
  }

  String get tierDescription {
    switch (tierLevel.toLowerCase()) {
      case 'bronze':
        return 'Earn 1x points on purchases';
      case 'silver':
        return 'Earn 1.5x points on purchases';
      case 'gold':
        return 'Earn 2x points on purchases';
      case 'platinum':
        return 'Earn 2.5x points on purchases';
      case 'diamond':
        return 'Earn 3x points on purchases';
      default:
        return 'Welcome to our loyalty program';
    }
  }
}

@freezed
class LoyaltyTransaction with _$LoyaltyTransaction {
  const factory LoyaltyTransaction({
    required int id,
    required int userId,
    int? orderId,
    required LoyaltyTransactionType transactionType,
    required int points,
    required int balanceBefore,
    required int balanceAfter,
    required String description,
    String? referenceId,
    DateTime? expiresAt,
    required DateTime createdAt,
    Order? order,
  }) = _LoyaltyTransaction;

  factory LoyaltyTransaction.fromJson(Map<String, dynamic> json) => 
      _$LoyaltyTransactionFromJson(json);

  const LoyaltyTransaction._();

  bool get isPositive => [
    LoyaltyTransactionType.earned,
    LoyaltyTransactionType.bonus,
    LoyaltyTransactionType.refunded,
  ].contains(transactionType);

  bool get isNegative => [
    LoyaltyTransactionType.spent,
    LoyaltyTransactionType.expired,
    LoyaltyTransactionType.penalty,
  ].contains(transactionType);

  Color get transactionColor {
    return isPositive ? Colors.green : Colors.red;
  }

  IconData get transactionIcon {
    switch (transactionType) {
      case LoyaltyTransactionType.earned:
        return Icons.add_circle;
      case LoyaltyTransactionType.spent:
        return Icons.remove_circle;
      case LoyaltyTransactionType.expired:
        return Icons.schedule;
      case LoyaltyTransactionType.refunded:
        return Icons.refresh;
      case LoyaltyTransactionType.bonus:
        return Icons.card_giftcard;
      case LoyaltyTransactionType.penalty:
        return Icons.warning;
    }
  }
}

@JsonEnum()
enum LoyaltyTransactionType {
  @JsonValue('earned')
  earned,
  @JsonValue('spent')
  spent,
  @JsonValue('expired')
  expired,
  @JsonValue('refunded')
  refunded,
  @JsonValue('bonus')
  bonus,
  @JsonValue('penalty')
  penalty,
}

// data/models/promotion_model.dart
@freezed
class Promotion with _$Promotion {
  const factory Promotion({
    required int id,
    int? adminId,
    required String name,
    String? description,
    required PromotionType promotionType,
    required double discountValue,
    int? buyQuantity,
    int? getQuantity,
    required String targetAudience,
    required String scopeType,
    double? minimumOrderAmount,
    double? maximumOrderAmount,
    double? maxDiscountAmount,
    int? usageLimit,
    int? usageLimitPerUser,
    required int currentUsage,
    DateTime? startDate,
    DateTime? endDate,
    bool? isRecurring,
    String? recurrencePattern,
    List<int>? recurrenceDays,
    Map<String, dynamic>? recurrenceData,
    int? priority,
    bool? isFeatured,
    bool? isStackable,
    bool? requiresCoupon,
    required String status,
    required DateTime createdAt,
    required DateTime updatedAt,
    List<Product>? products,
    List<String>? categories,
  }) = _Promotion;

  factory Promotion.fromJson(Map<String, dynamic> json) => _$PromotionFromJson(json);

  const Promotion._();

  bool get isActive {
    final now = DateTime.now();
    return status == 'active' &&
           (startDate == null || startDate!.isBefore(now)) &&
           (endDate == null || endDate!.isAfter(now)) &&
           (usageLimit == null || currentUsage < usageLimit!);
  }

  bool get hasUsageLimit => usageLimit != null && currentUsage >= usageLimit!;

  String get discountDisplayText {
    switch (promotionType) {
      case PromotionType.percentage:
        return '${discountValue.toStringAsFixed(0)}% OFF';
      case PromotionType.fixedAmount:
        return '\${discountValue.toStringAsFixed(2)} OFF';
      case PromotionType.buyXGetY:
        return 'Buy $buyQuantity Get $getQuantity';
      case PromotionType.freeShipping:
        return 'FREE SHIPPING';
      case PromotionType.bundle:
        return 'BUNDLE DEAL';
    }
  }

  Color get promotionColor {
    if (!isActive) return Colors.grey;
    switch (promotionType) {
      case PromotionType.percentage:
        return Colors.red;
      case PromotionType.fixedAmount:
        return Colors.orange;
      case PromotionType.buyXGetY:
        return Colors.purple;
      case PromotionType.freeShipping:
        return Colors.blue;
      case PromotionType.bundle:
        return Colors.green;
    }
  }

  double calculateDiscount(double orderAmount) {
    if (!isActive || hasUsageLimit) return 0.0;
    
    if (minimumOrderAmount != null && orderAmount < minimumOrderAmount!) {
      return 0.0;
    }

    if (maximumOrderAmount != null && orderAmount > maximumOrderAmount│   │   │   ├── input_styles.dart
│   │   │   └── color_schemes.dart
│   │   ├── errors/
│   │   │   ├── exceptions.dart
│   │   │   ├── failures.dart
│   │   │   └── error_handler.dart
│   │   └── network/
│   │       ├── network_info.dart
│   │       ├── dio_client.dart
│   │       └── interceptors.dart
│   ├── data/
│   │   ├── models/
│   │   │   ├── user_model.dart
│   │   │   ├── product_model.dart
│   │   │   ├── cart_model.dart
│   │   │   ├── order_model.dart
│   │   │   ├── promotion_model.dart
│   │   │   ├── coupon_model.dart
│   │   │   ├── loyalty_model.dart
│   │   │   ├── notification_model.dart
│   │   │   ├── wishlist_model.dart
│   │   │   ├── review_model.dart
│   │   │   ├── analytics_model.dart
│   │   │   └── response_model.dart
│   │   ├── repositories/
│   │   │   ├── auth_repository.dart
│   │   │   ├── product_repository.dart
│   │   │   ├── cart_repository.dart
│   │   │   ├── order_repository.dart
│   │   │   ├── promotion_repository.dart
│   │   │   ├── loyalty_repository.dart
│   │   │   ├── notification_repository.dart
│   │   │   └── analytics_repository.dart
│   │   └── datasources/
│   │       ├── remote/
│   │       │   ├── auth_remote_datasource.dart
│   │       │   ├── product_remote_datasource.dart
│   │       │   ├── cart_remote_datasource.dart
│   │       │   └── order_remote_datasource.dart
│   │       └── local/
│   │           ├── auth_local_datasource.dart
│   │           ├── cart_local_datasource.dart
│   │           └── settings_local_datasource.dart
│   ├── domain/
│   │   ├── entities/
│   │   │   ├── user.dart
│   │   │   ├── product.dart
│   │   │   ├── cart.dart
│   │   │   ├── order.dart
│   │   │   └── promotion.dart
│   │   ├── usecases/
│   │   │   ├── auth/
│   │   │   ├── product/
│   │   │   ├── cart/
│   │   │   ├── order/
│   │   │   └── loyalty/
│   │   └── repositories/
│   │       └── base_repository.dart
│   ├── presentation/
│   │   ├── providers/
│   │   │   ├── auth_provider.dart
│   │   │   ├── product_provider.dart
│   │   │   ├── cart_provider.dart
│   │   │   ├── order_provider.dart
│   │   │   ├── promotion_provider.dart
│   │   │   ├── loyalty_provider.dart
│   │   │   ├── notification_provider.dart
│   │   │   ├── wishlist_provider.dart
│   │   │   ├── theme_provider.dart
│   │   │   ├── language_provider.dart
│   │   │   └── connectivity_provider.dart
│   │   ├── screens/
│   │   │   ├── splash/
│   │   │   │   ├── splash_screen.dart
│   │   │   │   └── onboarding_screen.dart
│   │   │   ├── auth/
│   │   │   │   ├── login_screen.dart
│   │   │   │   ├── register_screen.dart
│   │   │   │   ├── forgot_password_screen.dart
│   │   │   │   ├── verify_email_screen.dart
│   │   │   │   └── reset_password_screen.dart
│   │   │   ├── owner/
│   │   │   │   ├── owner_dashboard.dart
│   │   │   │   ├── analytics_overview.dart
│   │   │   │   ├── commission_dashboard.dart
│   │   │   │   ├── user_management.dart
│   │   │   │   ├── system_settings.dart
│   │   │   │   └── reports_screen.dart
│   │   │   ├── admin/
│   │   │   │   ├── admin_dashboard.dart
│   │   │   │   ├── product_management/
│   │   │   │   │   ├── product_list_screen.dart
│   │   │   │   │   ├── add_product_screen.dart
│   │   │   │   │   ├── edit_product_screen.dart
│   │   │   │   │   ├── bulk_upload_screen.dart
│   │   │   │   │   └── inventory_screen.dart
│   │   │   │   ├── order_management/
│   │   │   │   │   ├── order_list_screen.dart
│   │   │   │   │   ├── order_detail_screen.dart
│   │   │   │   │   └── order_tracking_screen.dart
│   │   │   │   ├── promotion_management/
│   │   │   │   │   ├── promotion_list_screen.dart
│   │   │   │   │   ├── create_promotion_screen.dart
│   │   │   │   │   ├── edit_promotion_screen.dart
│   │   │   │   │   ├── coupon_management_screen.dart
│   │   │   │   │   └── promotion_analytics_screen.dart
│   │   │   │   ├── analytics/
│   │   │   │   │   ├── sales_dashboard.dart
│   │   │   │   │   ├── product_analytics.dart
│   │   │   │   │   ├── customer_analytics.dart
│   │   │   │   │   └── revenue_reports.dart
│   │   │   │   └── settings/
│   │   │   │       ├── admin_profile.dart
│   │   │   │       ├── business_settings.dart
│   │   │   │       └── notification_settings.dart
│   │   │   ├── supermarket/
│   │   │   │   ├── supermarket_dashboard.dart
│   │   │   │   ├── catalog/
│   │   │   │   │   ├── wholesale_catalog.dart
│   │   │   │   │   ├── product_detail_screen.dart
│   │   │   │   │   ├── category_screen.dart
│   │   │   │   │   └── search_screen.dart
│   │   │   │   ├── cart/
│   │   │   │   │   ├── cart_screen.dart
│   │   │   │   │   ├── checkout_screen.dart
│   │   │   │   │   └── order_summary_screen.dart
│   │   │   │   ├── orders/
│   │   │   │   │   ├── order_history_screen.dart
│   │   │   │   │   ├── order_detail_screen.dart
│   │   │   │   │   ├── reorder_screen.dart
│   │   │   │   │   └── order_tracking_screen.dart
│   │   │   │   ├── promotions/
│   │   │   │   │   ├── promotions_screen.dart
│   │   │   │   │   ├── flash_sales_screen.dart
│   │   │   │   │   └── bulk_discounts_screen.dart
│   │   │   │   └── account/
│   │   │   │       ├── profile_screen.dart
│   │   │   │       ├── business_profile_screen.dart
│   │   │   │       ├── address_management.dart
│   │   │   │       └── payment_methods.dart
│   │   │   ├── customer/
│   │   │   │   ├── customer_dashboard.dart
│   │   │   │   ├── catalog/
│   │   │   │   │   ├── retail_catalog.dart
│   │   │   │   │   ├── product_detail_screen.dart
│   │   │   │   │   ├── category_screen.dart
│   │   │   │   │   ├── search_screen.dart
│   │   │   │   │   └── featured_products.dart
│   │   │   │   ├── cart/
│   │   │   │   │   ├── cart_screen.dart
│   │   │   │   │   ├── checkout_screen.dart
│   │   │   │   │   └── order_summary_screen.dart
│   │   │   │   ├── orders/
│   │   │   │   │   ├── order_history_screen.dart
│   │   │   │   │   ├── order_detail_screen.dart
│   │   │   │   │   ├── reorder_screen.dart
│   │   │   │   │   └── order_tracking_screen.dart
│   │   │   │   ├── loyalty/
│   │   │   │   │   ├── loyalty_dashboard.dart
│   │   │   │   │   ├── points_history_screen.dart
│   │   │   │   │   ├── redeem_points_screen.dart
│   │   │   │   │   └── tier_benefits_screen.dart
│   │   │   │   ├── promotions/
│   │   │   │   │   ├── promotions_screen.dart
│   │   │   │   │   ├── coupons_screen.dart
│   │   │   │   │   ├── flash_sales_screen.dart
│   │   │   │   │   └── seasonal_offers_screen.dart
│   │   │   │   ├── wishlist/
│   │   │   │   │   ├── wishlist_screen.dart
│   │   │   │   │   └── wishlist_item_card.dart
│   │   │   │   ├── reviews/
│   │   │   │   │   ├── review_list_screen.dart
│   │   │   │   │   ├── write_review_screen.dart
│   │   │   │   │   └── review_detail_screen.dart
│   │   │   │   └── account/
│   │   │   │       ├── profile_screen.dart
│   │   │   │       ├── address_management.dart
│   │   │   │       ├── payment_methods.dart
│   │   │   │       └── preferences_screen.dart
│   │   │   └── shared/
│   │   │       ├── notifications_screen.dart
│   │   │       ├── settings_screen.dart
│   │   │       ├── help_support_screen.dart
│   │   │       ├── about_screen.dart
│   │   │       └── language_selection_screen.dart
│   │   ├── widgets/
│   │   │   ├── common/
│   │   │   │   ├── buttons/
│   │   │   │   │   ├── primary_button.dart
│   │   │   │   │   ├── secondary_button.dart
│   │   │   │   │   ├── icon_button.dart
│   │   │   │   │   └── floating_action_button.dart
│   │   │   │   ├── inputs/
│   │   │   │   │   ├── text_field.dart
│   │   │   │   │   ├── password_field.dart
│   │   │   │   │   ├── search_field.dart
│   │   │   │   │   ├── dropdown_field.dart
│   │   │   │   │   └── date_picker_field.dart
│   │   │   │   ├── cards/
│   │   │   │   │   ├── base_card.dart
│   │   │   │   │   ├── info_card.dart
│   │   │   │   │   ├── metric_card.dart
│   │   │   │   │   └── expansion_card.dart
│   │   │   │   ├── loaders/
│   │   │   │   │   ├── loading_spinner.dart
│   │   │   │   │   ├── skeleton_loader.dart
│   │   │   │   │   ├── shimmer_loader.dart
│   │   │   │   │   └── page_loader.dart
│   │   │   │   ├── dialogs/
│   │   │   │   │   ├── confirmation_dialog.dart
│   │   │   │   │   ├── error_dialog.dart
│   │   │   │   │   ├── success_dialog.dart
│   │   │   │   │   └── custom_dialog.dart
│   │   │   │   ├── navigation/
│   │   │   │   │   ├── app_bar.dart
│   │   │   │   │   ├── bottom_nav_bar.dart
│   │   │   │   │   ├── drawer.dart
│   │   │   │   │   └── tab_bar.dart
│   │   │   │   ├── images/
│   │   │   │   │   ├── cached_image.dart
│   │   │   │   │   ├── avatar_image.dart
│   │   │   │   │   ├── product_image.dart
│   │   │   │   │   └── image_carousel.dart
│   │   │   │   ├── lists/
│   │   │   │   │   ├── infinite_list.dart
│   │   │   │   │   ├── refresh_list.dart
│   │   │   │   │   ├── grid_view.dart
│   │   │   │   │   └── search_list.dart
│   │   │   │   └── misc/
│   │   │   │       ├── empty_state.dart
│   │   │   │       ├── error_state.dart
│   │   │   │       ├── badge.dart
│   │   │   │       ├── chip.dart
│   │   │   │       ├── divider.dart
│   │   │   │       ├── progress_bar.dart
│   │   │   │       └── rating_widget.dart
│   │   │   ├── product/
│   │   │   │   ├── product_card.dart
│   │   │   │   ├── product_grid.dart
│   │   │   │   ├── product_list_item.dart
│   │   │   │   ├── product_details_view.dart
│   │   │   │   ├── product_image_gallery.dart
│   │   │   │   ├── product_price_widget.dart
│   │   │   │   ├── stock_indicator.dart
│   │   │   │   ├── product_filter_widget.dart
│   │   │   │   ├── product_sort_widget.dart
│   │   │   │   └── related_products.dart
│   │   │   ├── cart/
│   │   │   │   ├── cart_item_widget.dart
│   │   │   │   ├── cart_summary_widget.dart
│   │   │   │   ├── quantity_selector.dart
│   │   │   │   ├── add_to_cart_button.dart
│   │   │   │   ├── cart_badge.dart
│   │   │   │   ├── coupon_input_widget.dart
│   │   │   │   ├── loyalty_points_widget.dart
│   │   │   │   └── shipping_calculator.dart
│   │   │   ├── order/
│   │   │   │   ├── order_card.dart
│   │   │   │   ├── order_status_widget.dart
│   │   │   │   ├── order_timeline.dart
│   │   │   │   ├── order_item_widget.dart
│   │   │   │   ├── order_summary_widget.dart
│   │   │   │   ├── delivery_info_widget.dart
│   │   │   │   └── tracking_widget.dart
│   │   │   ├── promotion/
│   │   │   │   ├── promotion_card.dart
│   │   │   │   ├── promotion_banner.dart
│   │   │   │   ├── coupon_card.dart
│   │   │   │   ├── flash_sale_timer.dart
│   │   │   │   ├── discount_badge.dart
│   │   │   │   └── promotion_carousel.dart
│   │   │   ├── loyalty/
│   │   │   │   ├── loyalty_card.dart
│   │   │   │   ├── points_balance_widget.dart
│   │   │   │   ├── tier_progress_widget.dart
│   │   │   │   ├── points_history_item.dart
│   │   │   │   └── redeem_points_widget.dart
│   │   │   ├── notification/
│   │   │   │   ├── notification_item.dart
│   │   │   │   ├── notification_badge.dart
│   │   │   │   ├── notification_list.dart
│   │   │   │   └── push_notification_handler.dart
│   │   │   ├── analytics/
│   │   │   │   ├── chart_widget.dart
│   │   │   │   ├── line_chart.dart
│   │   │   │   ├── bar_chart.dart
│   │   │   │   ├── pie_chart.dart
│   │   │   │   ├── metric_tile.dart
│   │   │   │   ├── revenue_chart.dart
│   │   │   │   ├── sales_trend_chart.dart
│   │   │   │   └── top_products_widget.dart
│   │   │   └── role_specific/
│   │   │       ├── owner/
│   │   │       │   ├── commission_overview.dart
│   │   │       │   ├── platform_stats.dart
│   │   │       │   ├── user_growth_chart.dart
│   │   │       │   └── revenue_breakdown.dart
│   │   │       ├── admin/
│   │   │       │   ├── inventory_widget.dart
│   │   │       │   ├── low_stock_alert.dart
│   │   │       │   ├── sales_summary.dart
│   │   │       │   ├── order_status_chart.dart
│   │   │       │   └── customer_insights.dart
│   │   │       ├── supermarket/
│   │   │       │   ├── bulk_order_widget.dart
│   │   │       │   ├── wholesale_pricing.dart
│   │   │       │   ├── volume_discounts.dart
│   │   │       │   └── reorder_suggestions.dart
│   │   │       └── customer/
│   │   │           ├── recommended_products.dart
│   │   │           ├── recent_orders.dart
│   │   │           ├── seasonal_offers.dart
│   │   │           └── personalized_deals.dart
│   │   └── navigation/
│   │       ├── app_router.dart
│   │       ├── route_generator.dart
│   │       ├── navigation_service.dart
│   │       └── route_guards.dart
│   ├── l10n/
│   │   ├── app_localizations.dart
│   │   ├── app_localizations_en.dart
│   │   ├── app_localizations_ar.dart
│   │   ├── app_en.arb
│   │   └── app_ar.arb
│   └── main.dart
├── assets/
│   ├── images/
│   │   ├── logos/
│   │   ├── icons/
│   │   ├── illustrations/
│   │   └── placeholders/
│   ├── fonts/
│   └── animations/
├── test/
│   ├── unit/
│   ├── widget/
│   └── integration/
├── integration_test/
├── android/
├── ios/
├── web/
├── pubspec.yaml
├── pubspec.lock
├── analysis_options.yaml
└── README.md
```

### **Step 13: Enhanced Flutter Dependencies**

```yaml
name: tradesuper_app
description: B2B/B2C Wholesale-Retail Platform

publish_to: 'none'
version: 1.0.0+1

environment:
  sdk: '>=3.0.0 <4.0.0'
  flutter: ">=3.10.0"

dependencies:
  flutter:
    sdk: flutter
  flutter_localizations:
    sdk: flutter

  # State Management
  provider: ^6.1.1
  flutter_bloc: ^8.1.3
  hydrated_bloc: ^9.1.2

  # Network & API
  dio: ^5.3.3
  retrofit: ^4.0.3
  pretty_dio_logger: ^1.3.1
  connectivity_plus: ^5.0.2

  # Local Storage
  shared_preferences: ^2.2.2
  flutter_secure_storage: ^9.0.0
  hive: ^2.2.3
  hive_flutter: ^1.1.0
  path_provider: ^2.1.1

  # UI Components & Animations
  cached_network_image: ^3.3.0
  image_picker: ^1.0.4
  photo_view: ^0.14.0
  carousel_slider: ^4.2.1
  shimmer: ^3.0.0
  lottie: ^2.7.0
  flutter_staggered_animations: ^1.1.1
  animations: ^2.0.8
  flutter_staggered_grid_view: ^0.7.0

  # Navigation & Routing
  go_router: ^12.1.1
  auto_route: ^7.8.4

  # Forms & Validation
  reactive_forms: ^16.1.1
  form_builder_validators: ^9.1.0

  # Charts & Analytics
  fl_chart: ^0.66.0
  syncfusion_flutter_charts: ^23.2.7

  # Date & Time
  intl: ^0.18.1
  timeago: ^3.6.0
  table_calendar: ^3.0.9

  # Media & Files
  file_picker: ^6.1.1
  image_cropper: ^5.0.1
  video_player: ^2.8.1
  permission_handler: ^11.1.0

  # Push Notifications
  firebase_core: ^2.24.2
  firebase_messaging: ^14.7.9
  firebase_analytics: ^10.7.4
  flutter_local_notifications: ^16.3.0

  # QR Code & Barcode
  qr_flutter: ^4.1.0
  qr_code_scanner: ^1.0.1
  mobile_scanner: ^3.5.6

  # Maps & Location
  geolocator: ^10.1.0
  geocoding: ^2.1.1
  google_maps_flutter: ^2.5.0

  # Utils & Helpers
  uuid: ^4.2.1
  logger: ^2.0.2
  url_launcher: ^6.2.2
  share_plus: ^7.2.2
  package_info_plus: ^5.0.1
  device_info_plus: ^9.1.1
  get_it: ^7.6.4
  injectable: ^2.3.2
  freezed_annotation: ^2.4.1
  json_annotation: ^4.8.1

  # Platform Specific
  cupertino_icons: ^1.0.6

dev_dependencies:
  flutter_test:
    sdk: flutter
  integration_test:
    sdk: flutter

  # Code Generation
  build_runner: ^2.4.7
  retrofit_generator: ^8.0.4
  hive_generator: ^2.0.1
  injectable_generator: ^2.4.1
  freezed: ^2.4.6
  json_serializable: ^6.7.1
  auto_route_generator: ^7.3.2

  # Testing
  mockito: ^5.4.4
  bloc_test: ^9.1.5
  mocktail: ^1.0.1

  # Linting & Analysis
  flutter_lints: ^3.0.0
  dart_code_metrics: ^5.7.6

flutter:
  uses-material-design: true
  generate: true

  assets:
    - assets/images/
    - assets/images/logos/
    - assets/images/icons/
    - assets/images/illustrations/
    - assets/images/placeholders/
    - assets/animations/

  fonts:
    - family: Inter
      fonts:
        - asset: assets/fonts/Inter-Regular.ttf
        - asset: assets/fonts/Inter-Medium.ttf
          weight: 500
        - asset: assets/fonts/Inter-SemiBold.ttf
          weight: 600
        - asset: assets/fonts/Inter-Bold.ttf
          weight: 700
    - family: Cairo
      fonts:
        - asset: assets/fonts/Cairo-Regular.ttf
        - asset: assets/fonts/Cairo-Medium.ttf
          weight: 500
        - asset: assets/fonts/Cairo-SemiBold.ttf
          weight: 600
        - asset: assets/fonts/Cairo-Bold.ttf
          weight: 700
```

## 2.2 Core Flutter Implementation

### **Step 14: API Service & Network Layer**

```dart
// core/network/dio_client.dart
import 'package:dio/dio.dart';
import 'package:flutter_secure_storage/flutter_secure_storage.dart';
import 'package:pretty_dio_logger/pretty_dio_logger.dart';
import '../constants/api_constants.dart';
import '../errors/exceptions.dart';

class DioClient {
  late Dio _dio;
  final FlutterSecureStorage _storage = const FlutterSecureStorage();

  DioClient() {
    _dio = Dio(BaseOptions(
      baseUrl: ApiConstants.baseUrl,
      connectTimeout: const Duration(seconds: 30),
      receiveTimeout: const Duration(seconds: 30),
      sendTimeout: const Duration(seconds: 30),
      headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
      },
    ));

    _setupInterceptors();
  }

  void _setupInterceptors() {
    // Auth interceptor
    _dio.interceptors.add(InterceptorsWrapper(
      onRequest: (options, handler) async {
        final token = await _storage.read(key: StorageKeys.authToken);
        if (token != null) {
          options.headers['Authorization'] = 'Bearer $token';
        }
        handler.next(options);
      },
      onError: (error, handler) async {
        if (error.response?.statusCode == 401) {
          await _handleUnauthorized();
        }
        handler.next(error);
      },
    ));

    // Logging interceptor (only in debug mode)
    if (kDebugMode) {
      _dio.interceptors.add(PrettyDioLogger(
        requestHeader: true,
        requestBody: true,
        responseHeader: false,
        responseBody: true,
        error: true,
        compact: true,
      ));
    }

    // Error handling interceptor
    _dio.interceptors.add(InterceptorsWrapper(
      onError: (error, handler) {
        final customError = _handleDioError(error);
        handler.reject(DioException(
          requestOptions: error.requestOptions,
          response: error.response,
          type: error.type,
          error: customError,
        ));
      },
    ));
  }

  Future<void> _handleUnauthorized() async {
    await _storage.delete(key: StorageKeys.authToken);
    await _storage.delete(key: StorageKeys.userRole);
    // Navigate to login screen
    // This would be handled by the auth provider
  }

  AppException _handleDioError(DioException error) {
    switch (error.type) {
      case DioExceptionType.connectionTimeout:
      case DioExceptionType.sendTimeout:
      case DioExceptionType.receiveTimeout:
        return NetworkException('Connection timeout. Please try again.');
      case DioExceptionType.badResponse:
        return _handleStatusCode(error.response!);
      case DioExceptionType.cancel:
        return NetworkException('Request was cancelled');
      case DioExceptionType.unknown:
        if (error.error.toString().contains('SocketException')) {
          return NetworkException('No internet connection');
        }
        return ServerException('Unexpected error occurred');
      default:
        return ServerException('Something went wrong');
    }
  }

  AppException _handleStatusCode(Response response) {
    final statusCode = response.statusCode;
    final data = response.data;

    switch (statusCode) {
      case 400:
        return ValidationException(
          data['message'] ?? 'Invalid request',
          data['errors'],
        );
      case 401:
        return AuthException('Unauthorized access');
      case 403:
        return AuthException('Access forbidden');
      case 404:
        return NotFoundException('Resource not found');
      case 422:
        return ValidationException(
          data['message'] ?? 'Validation failed',
          data['errors'],
        );
      case 429:
        return NetworkException('Too many requests. Please try again later.');
      case 500:
        return ServerException('Internal server error');
      case 502:
      case 503:
      case 504:
        return ServerException('Server temporarily unavailable');
      default:
        return ServerException('Server error: $statusCode');
    }
  }

  // HTTP Methods
  Future<Response> get(
    String path, {
    Map<String, dynamic>? queryParameters,
    Options? options,
    CancelToken? cancelToken,
  }) async {
    try {
      return await _dio.get(
        path,
        queryParameters: queryParameters,
        options: options,
        cancelToken: cancelToken,
      );
    } on DioException {
      rethrow;
    }
  }

  Future<Response> post(
    String path, {
    dynamic data,
    Map<String, dynamic>? queryParameters,
    Options? options,
    CancelToken? cancelToken,
  }) async {
    try {
      return await _dio.post(
        path,
        data: data,
        queryParameters: queryParameters,
        options: options,
        cancelToken: cancelToken,
      );100) : 0,
        tierDistribution: tierDistribution.map(tier => ({
          tier: tier.tierLevel,
          count: parseInt(tier.dataValues.count)
        })),
        recentActivity: recentActivity.map(activity => ({
          type: activity.transactionType,
          count: parseInt(activity.dataValues.count),
          totalPoints: parseInt(activity.dataValues.totalPoints || 0)
        })),
        topCustomers: topCustomers.map(customer => ({
          ...customer.toJSON(),
          pointsValue: customer.currentBalance * 0.01
        }))
      }
    });
  } catch (error) {
    console.error('Get loyalty overview error:', error);
    res.status(500).json({
      success: false,
      message: 'Error fetching loyalty overview'
    });
  }
};

// Cron job function to expire points
exports.expirePoints = async () => {
  try {
    const now = new Date();
    
    // Find all transactions with expired points
    const expiredTransactions = await LoyaltyTransaction.findAll({
      where: {
        transactionType: 'earned',
        expiresAt: { [Op.lt]: now }
      },
      include: [{
        model: User,
        include: [LoyaltyPoints]
      }]
    });

    for (const transaction of expiredTransactions) {
      const userId = transaction.userId;
      const pointsToExpire = transaction.points;
      
      const loyaltyAccount = await LoyaltyPoints.findOne({
        where: { userId }
      });

      if (!loyaltyAccount) continue;

      const balanceBefore = loyaltyAccount.currentBalance;
      const newBalance = Math.max(0, balanceBefore - pointsToExpire);
      const actualExpired = balanceBefore - newBalance;

      if (actualExpired > 0) {
        // Update loyalty account
        await loyaltyAccount.update({
          pointsExpired: loyaltyAccount.pointsExpired + actualExpired,
          currentBalance: newBalance
        });

        // Create expiration transaction
        await LoyaltyTransaction.create({
          userId,
          transactionType: 'expired',
          points: actualExpired,
          balanceBefore,
          balanceAfter: newBalance,
          description: `Points expired from order earned on ${transaction.createdAt.toDateString()}`,
          referenceId: `EXP-${transaction.id}`
        });

        // Mark original transaction as processed
        await transaction.update({ expiresAt: null });

        console.log(`Expired ${actualExpired} points for user ${userId}`);
      }
    }
  } catch (error) {
    console.error('Error expiring points:', error);
  }
};
```

### **Step 11: Order Management System**

#### **Enhanced Order Models & Controller**
```javascript
// models/Order.js (Enhanced)
const { DataTypes } = require('sequelize');
const sequelize = require('../config/database');

const Order = sequelize.define('Order', {
  orderNumber: {
    type: DataTypes.STRING(50),
    allowNull: false,
    unique: true,
    field: 'order_number'
  },
  buyerId: {
    type: DataTypes.INTEGER,
    allowNull: false,
    references: { model: 'users', key: 'id' },
    field: 'buyer_id'
  },
  adminId: {
    type: DataTypes.INTEGER,
    allowNull: false,
    references: { model: 'users', key: 'id' },
    field: 'admin_id'
  },
  orderType: {
    type: DataTypes.ENUM('B2B', 'B2C'),
    allowNull: false,
    field: 'order_type'
  },
  subtotal: {
    type: DataTypes.DECIMAL(10, 2),
    allowNull: false
  },
  discountAmount: {
    type: DataTypes.DECIMAL(10, 2),
    defaultValue: 0,
    field: 'discount_amount'
  },
  couponDiscount: {
    type: DataTypes.DECIMAL(10, 2),
    defaultValue: 0,
    field: 'coupon_discount'
  },
  loyaltyPointsUsed: {
    type: DataTypes.INTEGER,
    defaultValue: 0,
    field: 'loyalty_points_used'
  },
  loyaltyDiscount: {
    type: DataTypes.DECIMAL(10, 2),
    defaultValue: 0,
    field: 'loyalty_discount'
  },
  shippingCost: {
    type: DataTypes.DECIMAL(10, 2),
    defaultValue: 0,
    field: 'shipping_cost'
  },
  taxAmount: {
    type: DataTypes.DECIMAL(10, 2),
    defaultValue: 0,
    field: 'tax_amount'
  },
  totalAmount: {
    type: DataTypes.DECIMAL(10, 2),
    allowNull: false,
    field: 'total_amount'
  },
  commissionRate: {
    type: DataTypes.DECIMAL(5, 4),
    allowNull: false,
    field: 'commission_rate'
  },
  commissionAmount: {
    type: DataTypes.DECIMAL(10, 2),
    allowNull: false,
    field: 'commission_amount'
  },
  status: {
    type: DataTypes.ENUM('pending', 'confirmed', 'processing', 'shipped', 'delivered', 'cancelled', 'refunded'),
    defaultValue: 'pending'
  },
  paymentStatus: {
    type: DataTypes.ENUM('pending', 'paid', 'failed', 'refunded', 'partial'),
    defaultValue: 'pending',
    field: 'payment_status'
  },
  shippingAddress: {
    type: DataTypes.JSONB,
    field: 'shipping_address'
  },
  billingAddress: {
    type: DataTypes.JSONB,
    field: 'billing_address'
  },
  paymentMethod: {
    type: DataTypes.STRING(50),
    field: 'payment_method'
  },
  paymentReference: {
    type: DataTypes.STRING(255),
    field: 'payment_reference'
  },
  promotionId: {
    type: DataTypes.INTEGER,
    references: { model: 'promotions', key: 'id' },
    field: 'promotion_id'
  },
  couponId: {
    type: DataTypes.INTEGER,
    references: { model: 'coupons', key: 'id' },
    field: 'coupon_id'
  },
  deliveryDate: {
    type: DataTypes.DATEONLY,
    field: 'delivery_date'
  },
  deliveryTimeSlot: {
    type: DataTypes.STRING(50),
    field: 'delivery_time_slot'
  },
  trackingNumber: {
    type: DataTypes.STRING(100),
    field: 'tracking_number'
  },
  notes: DataTypes.TEXT,
  internalNotes: {
    type: DataTypes.TEXT,
    field: 'internal_notes'
  },
  cancelledReason: {
    type: DataTypes.TEXT,
    field: 'cancelled_reason'
  },
  refundAmount: {
    type: DataTypes.DECIMAL(10, 2),
    field: 'refund_amount'
  }
}, {
  tableName: 'orders',
  timestamps: true,
  createdAt: 'created_at',
  updatedAt: 'updated_at',
  hooks: {
    beforeCreate: async (order) => {
      if (!order.orderNumber) {
        const timestamp = Date.now();
        const random = Math.random().toString(36).substr(2, 6).toUpperCase();
        order.orderNumber = `ORD-${timestamp}-${random}`;
      }
    }
  }
});

// Instance methods
Order.prototype.canBeCancelled = function() {
  return ['pending', 'confirmed'].includes(this.status);
};

Order.prototype.canBeRefunded = function() {
  return ['delivered'].includes(this.status) && this.paymentStatus === 'paid';
};

Order.prototype.getTotalSavings = function() {
  return parseFloat(this.discountAmount) + 
         parseFloat(this.couponDiscount) + 
         parseFloat(this.loyaltyDiscount);
};

module.exports = Order;

// controllers/orderController.js
const Order = require('../models/Order');
const OrderItem = require('../models/OrderItem');
const Product = require('../models/Product');
const User = require('../models/User');
const Cart = require('../models/Cart');
const CartItem = require('../models/CartItem');
const Coupon = require('../models/Coupon');
const CouponUsage = require('../models/CouponUsage');
const Commission = require('../models/Commission');
const loyaltyController = require('./loyaltyController');
const promotionService = require('../services/promotionService');
const notificationService = require('../services/notificationService');
const emailService = require('../services/emailService');
const stockService = require('../services/stockService');

exports.createOrder = async (req, res) => {
  const transaction = await sequelize.transaction();
  
  try {
    const userId = req.user.id;
    const {
      shippingAddress, billingAddress, paymentMethod,
      deliveryDate, deliveryTimeSlot, notes,
      couponCode, loyaltyPointsUsed = 0
    } = req.body;

    const userType = req.user.role === 'supermarket' ? 'B2B' : 'B2C';

    // Get user's cart
    const cart = await Cart.findOne({
      where: { userId },
      include: [{
        model: CartItem,
        include: [{
          model: Product,
          where: { isActive: true }
        }]
      }],
      transaction
    });

    if (!cart || !cart.CartItems.length) {
      await transaction.rollback();
      return res.status(400).json({
        success: false,
        message: 'Cart is empty'
      });
    }

    // Validate stock availability
    for (const item of cart.CartItems) {
      if (!item.Product) {
        await transaction.rollback();
        return res.status(400).json({
          success: false,
          message: `Product in cart is no longer available`
        });
      }

      if (item.Product.stockQuantity < item.quantity) {
        await transaction.rollback();
        return res.status(400).json({
          success: false,
          message: `Insufficient stock for ${item.Product.name}. Only ${item.Product.stockQuantity} available.`
        });
      }
    }

    // Calculate order totals
    let subtotal = 0;
    const adminId = cart.CartItems[0].Product.adminId;

    // Ensure all items are from the same admin
    for (const item of cart.CartItems) {
      if (item.Product.adminId !== adminId) {
        await transaction.rollback();
        return res.status(400).json({
          success: false,
          message: 'All items must be from the same seller'
        });
      }
      subtotal += parseFloat(item.unitPrice) * item.quantity;
    }

    // Apply promotions
    let discountAmount = 0;
    let promotionId = null;
    
    const applicablePromotions = await promotionService.getApplicablePromotions({
      userId,
      userType,
      cartItems: cart.CartItems,
      subtotal
    });

    if (applicablePromotions.length > 0) {
      const bestPromotion = applicablePromotions[0]; // Highest priority
      discountAmount = await promotionService.calculateDiscount(bestPromotion, cart.CartItems, subtotal);
      promotionId = bestPromotion.id;
    }

    // Apply coupon
    let couponDiscount = 0;
    let couponId = null;
    
    if (couponCode) {
      const coupon = await Coupon.findOne({
        where: { 
          code: couponCode.toUpperCase(),
          isActive: true 
        },
        transaction
      });

      if (coupon && coupon.isValid()) {
        const userUsageCount = await CouponUsage.count({
          where: { couponId: coupon.id, userId },
          transaction
        });

        if (coupon.canBeUsedBy(userId, userUsageCount)) {
          couponDiscount = coupon.calculateDiscount(subtotal);
          couponId = coupon.id;
        }
      }
    }

    // Apply loyalty points
    let loyaltyDiscount = 0;
    if (loyaltyPointsUsed > 0 && userType === 'B2C') {
      const loyaltyAccount = await LoyaltyPoints.findOne({
        where: { userId },
        transaction
      });

      if (loyaltyAccount && loyaltyAccount.currentBalance >= loyaltyPointsUsed) {
        loyaltyDiscount = loyaltyPointsUsed * 0.01; // 1 point = $0.01
      } else {
        await transaction.rollback();
        return res.status(400).json({
          success: false,
          message: 'Insufficient loyalty points'
        });
      }
    }

    // Calculate shipping and tax
    const shippingCost = await calculateShippingCost(userType, subtotal, shippingAddress);
    const taxAmount = await calculateTax(subtotal, shippingAddress);

    // Calculate total
    const totalAmount = subtotal - discountAmount - couponDiscount - loyaltyDiscount + shippingCost + taxAmount;

    if (totalAmount < 0) {
      await transaction.rollback();
      return res.status(400).json({
        success: false,
        message: 'Invalid order total'
      });
    }

    // Calculate commission
    const commissionRate = userType === 'B2C' ? 
      parseFloat(process.env.B2C_COMMISSION_RATE || 0.07) : 
      parseFloat(process.env.B2B_COMMISSION_RATE || 0.03);
    const commissionAmount = totalAmount * commissionRate;

    // Create order
    const order = await Order.create({
      buyerId: userId,
      adminId,
      orderType: userType,
      subtotal,
      discountAmount,
      couponDiscount,
      loyaltyPointsUsed,
      loyaltyDiscount,
      shippingCost,
      taxAmount,
      totalAmount,
      commissionRate,
      commissionAmount,
      shippingAddress,
      billingAddress: billingAddress || shippingAddress,
      paymentMethod,
      promotionId,
      couponId,
      deliveryDate,
      deliveryTimeSlot,
      notes
    }, { transaction });

    // Create order items and update stock
    for (const cartItem of cart.CartItems) {
      await OrderItem.create({
        orderId: order.id,
        productId: cartItem.productId,
        productName: cartItem.Product.name,
        productSku: cartItem.Product.sku,
        quantity: cartItem.quantity,
        unitPrice: cartItem.unitPrice,
        totalPrice: cartItem.unitPrice * cartItem.quantity
      }, { transaction });

      // Update product stock
      await cartItem.Product.update({
        stockQuantity: cartItem.Product.stockQuantity - cartItem.quantity
      }, { transaction });

      // Log inventory movement
      await stockService.logInventoryMovement({
        productId: cartItem.productId,
        movementType: 'out',
        quantity: cartItem.quantity,
        quantityBefore: cartItem.Product.stockQuantity,
        quantityAfter: cartItem.Product.stockQuantity - cartItem.quantity,
        referenceType: 'order',
        referenceId: order.id,
        createdBy: userId
      }, { transaction });
    }

    // Create commission record
    await Commission.create({
      orderId: order.id,
      ownerId: 1, // Assuming owner has ID 1
      adminId,
      commissionAmount,
      commissionRate,
      orderType: userType
    }, { transaction });

    // Update coupon usage
    if (couponId) {
      await CouponUsage.create({
        couponId,
        userId,
        orderId: order.id,
        discountAmount: couponDiscount
      }, { transaction });

      await Coupon.increment('currentUsage', {
        where: { id: couponId },
        transaction
      });
    }

    // Update loyalty points
    if (loyaltyPointsUsed > 0) {
      const loyaltyAccount = await LoyaltyPoints.findOne({
        where: { userId },
        transaction
      });

      const balanceBefore = loyaltyAccount.currentBalance;
      await loyaltyAccount.update({
        pointsSpent: loyaltyAccount.pointsSpent + loyaltyPointsUsed,
        currentBalance: loyaltyAccount.currentBalance - loyaltyPointsUsed
      }, { transaction });

      await LoyaltyTransaction.create({
        userId,
        orderId: order.id,
        transactionType: 'spent',
        points: loyaltyPointsUsed,
        balanceBefore,
        balanceAfter: loyaltyAccount.currentBalance,
        description: `Points used for order #${order.orderNumber}`
      }, { transaction });
    }

    // Clear cart
    await CartItem.destroy({
      where: { cartId: cart.id },
      transaction
    });

    await transaction.commit();

    // Send notifications (after commit)
    await notificationService.sendOrderConfirmation(order);
    await emailService.sendOrderConfirmationEmail(order);

    // Award loyalty points (B2C only)
    if (userType === 'B2C') {
      await loyaltyController.awardPoints(order);
    }

    res.status(201).json({
      success: true,
      message: 'Order created successfully',
      order: {
        id: order.id,
        orderNumber: order.orderNumber,
        totalAmount: order.totalAmount,
        status: order.status,
        estimatedDelivery: order.deliveryDate
      }
    });
  } catch (error) {
    await transaction.rollback();
    console.error('Create order error:', error);
    res.status(500).json({
      success: false,
      message: 'Error creating order'
    });
  }
};

exports.getOrders = async (req, res) => {
  try {
    const {
      page = 1,
      limit = 20,
      status,
      orderType,
      startDate,
      endDate,
      search
    } = req.query;
    
    const offset = (page - 1) * limit;
    let whereClause = {};

    // Role-based filtering
    if (req.user.role === 'admin') {
      whereClause.adminId = req.user.id;
    } else if (['supermarket', 'customer'].includes(req.user.role)) {
      whereClause.buyerId = req.user.id;
    }
    // Owner can see all orders

    if (status) whereClause.status = status;
    if (orderType) whereClause.orderType = orderType;

    if (startDate || endDate) {
      whereClause.createdAt = {};
      if (startDate) whereClause.createdAt[Op.gte] = new Date(startDate);
      if (endDate) whereClause.createdAt[Op.lte] = new Date(endDate);
    }

    if (search) {
      whereClause[Op.or] = [
        { orderNumber: { [Op.iLike]: `%${search}%` } },
        { trackingNumber: { [Op.iLike]: `%${search}%` } }
      ];
    }

    const orders = await Order.findAndCountAll({
      where: whereClause,
      limit: parseInt(limit),
      offset: parseInt(offset),
      order: [['createdAt', 'DESC']],
      include: [
        {
          model: User,
          as: 'buyer',
          attributes: ['id', 'fullName', 'companyName', 'email']
        },
        {
          model: User,
          as: 'admin',
          attributes: ['id', 'fullName', 'companyName']
        },
        {
          model: OrderItem,
          include: [{
            model: Product,
            attributes: ['id', 'name', 'featuredImageUrl']
          }]
        }
      ]
    });

    // Add computed fields
    const ordersWithComputed = orders.rows.map(order => ({
      ...order.toJSON(),
      totalSavings: order.getTotalSavings(),
      canBeCancelled: order.canBeCancelled(),
      canBeRefunded: order.canBeRefunded()
    }));

    res.json({
      success: true,
      orders: ordersWithComputed,
      pagination: {
        currentPage: parseInt(page),
        totalPages: Math.ceil(orders.count / limit),
        totalItems: orders.count,
        itemsPerPage: parseInt(limit)
      }
    });
  } catch (error) {
    console.error('Get orders error:', error);
    res.status(500).json({
      success: false,
      message: 'Error fetching orders'
    });
  }
};

exports.getOrder = async (req, res) => {
  try {
    const { id } = req.params;
    let whereClause = { id };

    // Role-based access control
    if (req.user.role === 'admin') {
      whereClause.adminId = req.user.id;
    } else if (['supermarket', 'customer'].includes(req.user.role)) {
      whereClause.buyerId = req.user.id;
    }

    const order = await Order.findOne({
      where: whereClause,
      include: [
        {
          model: User,
          as: 'buyer',
          attributes: ['id', 'fullName', 'companyName', 'email', 'phone']
        },
        {
          model: User,
          as: 'admin',
          attributes: ['id', 'fullName', 'companyName', 'email', 'phone']
        },
        {
          model: OrderItem,
          include: [{
            model: Product,
            attributes: ['id', 'name', 'featuredImageUrl', 'category']
          }]
        },
        {
          model: Promotion,
          attributes: ['id', 'name', 'description']
        },
        {
          model: Coupon,
          attributes: ['id', 'code', 'name']
        }
      ]
    });

    if (!order) {
      return res.status(404).json({
        success: false,
        message: 'Order not found'
      });
    }

    res.json({
      success: true,
      order: {
        ...order.toJSON(),
        totalSavings: order.getTotalSavings(),
        canBeCancelled: order.canBeCancelled(),
        canBeRefunded: order.canBeRefunded()
      }
    });
  } catch (error) {
    console.error('Get order error:', error);
    res.status(500).json({
      success: false,
      message: 'Error fetching order'
    });
  }
};

exports.updateOrderStatus = async (req, res) => {
  try {
    const { id } = req.params;
    const { status, trackingNumber, internalNotes } = req.body;

    let whereClause = { id };
    
    // Only admin can update their orders
    if (req.user.role === 'admin') {
      whereClause.adminId = req.user.id;
    } else if (req.user.role !== 'owner') {
      return res.status(403).json({
        success: false,
        message: 'Not authorized to update order status'
      });
    }

    const order = await Order.findOne({
      where: whereClause,
      include: [{ model: User, as: 'buyer' }]
    });

    if (!order) {
      return res.status(404).json({
        success: false,
        message: 'Order not found'
      });
    }

    const oldStatus = order.status;
    
    await order.update({
      status,
      trackingNumber: trackingNumber || order.trackingNumber,
      internalNotes: internalNotes || order.internalNotes
    });

    // Send status update notification
    await notificationService.sendOrderStatusUpdate(order, oldStatus);
    
    // Award loyalty points when order is delivered (B2C only)
    if (status === 'delivered' && oldStatus !== 'delivered' && order.orderType === 'B2C') {
      await loyaltyController.awardPoints(order);
    }

    res.json({
      success: true,
      message: 'Order status updated successfully',
      order
    });
  } catch (error) {
    console.error('Update order status error:', error);
    res.status(500).json({
      success: false,
      message: 'Error updating order status'
    });
  }
};

exports.cancelOrder = async (req, res) => {
  const transaction = await sequelize.transaction();
  
  try {
    const { id } = req.params;
    const { reason } = req.body;

    let whereClause = { id };
    
    // Customers/Supermarkets can only cancel their own orders
    if (['supermarket', 'customer'].includes(req.user.role)) {
      whereClause.buyerId = req.user.id;
    } else if (req.user.role === 'admin') {
      whereClause.adminId = req.user.id;
    }

    const order = await Order.findOne({
      where: whereClause,
      include: [{ model: OrderItem, include: [Product] }],
      transaction
    });

    if (!order) {
      await transaction.rollback();
      return res.status(404).json({
        success: false,
        message: 'Order not found'
      });
    }

    if (!order.canBeCancelled()) {
      await transaction.rollback();
      return res.status(400).json({
        success: false,
        message: 'Order cannot be cancelled at this stage'
      });
    }

    // Restore stock for all items
    for (const item of order.OrderItems) {
      if (item.Product) {
        await item.Product.update({
          stockQuantity: item.Product.stockQuantity + item.quantity
        }, { transaction });

        // Log inventory movement
        await stockService.logInventoryMovement({
          productId: item.productId,
          movementType: 'in',
          quantity: item.quantity,
          quantityBefore: item.Product.stockQuantity,
          quantityAfter: item.Product.stockQuantity + item.quantity,
          referenceType: 'order_cancellation',
          referenceId: order.id,
          createdBy: req.user.id,
          notes: `Order ${order.orderNumber} cancelled`
        }, { transaction });
      }
    }

    // Restore loyalty points if used
    if (order.loyaltyPointsUsed > 0) {
      const loyaltyAccount = await LoyaltyPoints.findOne({
        where: { userId: order.buyerId },
        transaction
      });

      if (loyaltyAccount) {
        const balanceBefore = loyaltyAccount.currentBalance;
        await loyaltyAccount.update({
          pointsSpent: loyaltyAccount.pointsSpent - order.loyaltyPointsUsed,
          currentBalance: loyaltyAccount.currentBalance + order.loyaltyPointsUsed
        }, { transaction });

        await LoyaltyTransaction.create({
          userId: order.buyerId,
          orderId: order.id,
          transactionType: 'refunded',
          points: order.loyaltyPointsUsed,
          balanceBefore,
          balanceAfter: loyaltyAccount.currentBalance,
          description: `Points refunded from cancelled order #${order.orderNumber}`
        }, { transaction });
      }
    }

    // Update order status
    await order.update({
      status: 'cancelled',
      cancelledReason: reason,
      paymentStatus: order.paymentStatus === 'paid' ? 'refunded' : order.paymentStatus
    }, { transaction });

    await transaction.commit();

    // Send cancellation notification
    await notificationService.sendOrderCancellation(order, reason);

    res.json({
      success: true,
      message: 'Order cancelled successfully',
      order
    });
  } catch (error) {
    await transaction.rollback();
    console.error('Cancel order error:', error);
    res.status(500).json({
      success: false,
      message: 'Error cancelling order'
    });
  }
};

// Helper functions
const calculateShippingCost = async (userType, subtotal, address) => {
  // Implement shipping cost calculation logic
  if (userType === 'B2B' && subtotal > 500) return 0; // Free shipping for large B2B orders
  if (userType === 'B2C' && subtotal > 100) return 0; // Free shipping threshold
  
  // Basic shipping rates
  return userType === 'B2B' ? 25.00 : 15.00;
};

const calculateTax = async (subtotal, address) => {
  // Implement tax calculation based on location
  const taxRate = 0.10; // 10% default tax rate
  return subtotal * taxRate;
};
```

---

# Phase 2: Frontend Development

## 2.1 Flutter Project Setup & Architecture

### **Step 12: Enhanced Flutter Project Structure**

```
tradesuper_app/
├── lib/
│   ├── core/
│   │   ├── constants/
│   │   │   ├── api_constants.dart
│   │   │   ├── app_colors.dart
│   │   │   ├── app_strings.dart
│   │   │   ├── app_routes.dart
│   │   │   ├── asset_constants.dart
│   │   │   └── storage_keys.dart
│   │   ├── services/
│   │   │   ├── api_service.dart
│   │   │   ├── auth_service.dart
│   │   │   ├── storage_service.dart
│   │   │   ├── notification_service.dart
│   │   │   ├── connectivity_service.dart
│   │   │   ├── image_service.dart
│   │   │   ├── location_service.dart
│   │   │   └── analytics_service.dart
│   │   ├── utils/
│   │   │   ├── validators.dart
│   │   │   ├── formatters.dart
│   │   │   ├── date_utils.dart
│   │   │   ├── currency_utils.dart
│   │   │   ├── image_utils.dart
│   │   │   ├── file_utils.dart
│   │   │   └── platform_utils.dart
│   │   ├── theme/
│   │   │   ├── app_theme.dart
│   │   │   ├── text_styles.dart
│   │   │   ├── button_styles.dart
│        validUntil: coupon.validUntil
      },
      discountAmount,
      applicableAmount,
      usageCount: userUsageCount,
      remainingUses: coupon.usageLimitPerUser ? coupon.usageLimitPerUser - userUsageCount : null
    });
  } catch (error) {
    console.error('Validate coupon error:', error);
    res.status(500).json({
      success: false,
      message: 'Error validating coupon'
    });
  }
};

exports.getUserCoupons = async (req, res) => {
  try {
    const userId = req.user.id;
    const userType = req.user.role === 'supermarket' ? 'B2B' : 'B2C';
    const { orderAmount } = req.query;
    const now = new Date();

    const coupons = await Coupon.findAll({
      where: {
        isActive: true,
        [Op.or]: [
          { targetAudience: 'both' },
          { targetAudience: userType }
        ],
        [Op.or]: [
          { validFrom: { [Op.lte]: now } },
          { validFrom: { [Op.is]: null } }
        ],
        [Op.or]: [
          { validUntil: { [Op.gte]: now } },
          { validUntil: { [Op.is]: null } }
        ],
        [Op.or]: [
          { usageLimit: { [Op.gt]: sequelize.col('current_usage') } },
          { usageLimit: { [Op.is]: null } }
        ]
      },
      attributes: [
        'id', 'code', 'name', 'description', 'discountType', 
        'discountValue', 'minimumOrderAmount', 'maxDiscountAmount',
        'validUntil', 'usageLimitPerUser'
      ]
    });

    // Filter out coupons that user has reached usage limit
    const availableCoupons = [];
    
    for (const coupon of coupons) {
      const userUsageCount = await CouponUsage.count({
        where: { couponId: coupon.id, userId }
      });

      if (coupon.usageLimitPerUser && userUsageCount >= coupon.usageLimitPerUser) {
        continue;
      }

      // Check first order only restriction
      if (coupon.firstOrderOnly) {
        const orderCount = await Order.count({
          where: { buyerId: userId, status: { [Op.ne]: 'cancelled' } }
        });
        if (orderCount > 0) continue;
      }

      // Calculate potential discount if order amount provided
      let potentialDiscount = null;
      if (orderAmount) {
        potentialDiscount = coupon.calculateDiscount(parseFloat(orderAmount));
      }

      availableCoupons.push({
        ...coupon.toJSON(),
        userUsageCount,
        remainingUses: coupon.usageLimitPerUser ? coupon.usageLimitPerUser - userUsageCount : null,
        potentialDiscount
      });
    }

    res.json({
      success: true,
      coupons: availableCoupons
    });
  } catch (error) {
    console.error('Get user coupons error:', error);
    res.status(500).json({
      success: false,
      message: 'Error fetching available coupons'
    });
  }
};

exports.applyCouponToCart = async (req, res) => {
  try {
    const { couponCode } = req.body;
    const userId = req.user.id;

    // Get user's cart with items
    const cart = await Cart.findOne({
      where: { userId },
      include: [{
        model: CartItem,
        include: [Product]
      }]
    });

    if (!cart || !cart.CartItems.length) {
      return res.status(400).json({
        success: false,
        message: 'Cart is empty'
      });
    }

    // Calculate cart totals and prepare items for validation
    let orderAmount = 0;
    const cartItems = cart.CartItems.map(item => {
      const itemTotal = item.unitPrice * item.quantity;
      orderAmount += itemTotal;
      return {
        productId: item.productId,
        category: item.Product.category,
        unitPrice: item.unitPrice,
        quantity: item.quantity,
        total: itemTotal
      };
    });

    // Validate coupon
    const validationResult = await this.validateCoupon({
      body: { code: couponCode, orderAmount, cartItems },
      user: req.user
    }, {
      json: (data) => data,
      status: (code) => ({ json: (data) => ({ statusCode: code, ...data }) })
    });

    if (validationResult.statusCode && validationResult.statusCode !== 200) {
      return res.status(validationResult.statusCode).json(validationResult);
    }

    res.json({
      success: true,
      message: 'Coupon applied successfully',
      ...validationResult
    });
  } catch (error) {
    console.error('Apply coupon error:', error);
    res.status(500).json({
      success: false,
      message: 'Error applying coupon'
    });
  }
};

exports.getAdminCoupons = async (req, res) => {
  try {
    const { page = 1, limit = 20, status, search } = req.query;
    const offset = (page - 1) * limit;

    let whereClause = { adminId: req.user.id };
    
    if (status) {
      if (status === 'active') {
        const now = new Date();
        whereClause[Op.and] = [
          { isActive: true },
          {
            [Op.or]: [
              { validFrom: { [Op.lte]: now } },
              { validFrom: { [Op.is]: null } }
            ]
          },
          {
            [Op.or]: [
              { validUntil: { [Op.gte]: now } },
              { validUntil: { [Op.is]: null } }
            ]
          }
        ];
      } else if (status === 'expired') {
        whereClause.validUntil = { [Op.lt]: new Date() };
      } else if (status === 'inactive') {
        whereClause.isActive = false;
      }
    }

    if (search) {
      whereClause[Op.or] = [
        { code: { [Op.iLike]: `%${search}%` } },
        { name: { [Op.iLike]: `%${search}%` } }
      ];
    }

    const coupons = await Coupon.findAndCountAll({
      where: whereClause,
      limit: parseInt(limit),
      offset: parseInt(offset),
      order: [['createdAt', 'DESC']],
      include: [{
        model: CouponUsage,
        attributes: ['id'],
        required: false
      }]
    });

    // Add usage statistics
    const couponsWithStats = await Promise.all(coupons.rows.map(async (coupon) => {
      const totalUsage = await CouponUsage.count({
        where: { couponId: coupon.id }
      });

      const recentUsage = await CouponUsage.count({
        where: {
          couponId: coupon.id,
          usedAt: { [Op.gte]: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000) } // Last 30 days
        }
      });

      return {
        ...coupon.toJSON(),
        totalUsage,
        recentUsage,
        usageRate: coupon.usageLimit ? (totalUsage / coupon.usageLimit * 100) : null
      };
    }));

    res.json({
      success: true,
      coupons: couponsWithStats,
      pagination: {
        currentPage: parseInt(page),
        totalPages: Math.ceil(coupons.count / limit),
        totalItems: coupons.count,
        itemsPerPage: parseInt(limit)
      }
    });
  } catch (error) {
    console.error('Get admin coupons error:', error);
    res.status(500).json({
      success: false,
      message: 'Error fetching coupons'
    });
  }
};

exports.updateCoupon = async (req, res) => {
  try {
    const { id } = req.params;
    const updates = req.body;

    const coupon = await Coupon.findOne({
      where: { id, adminId: req.user.id }
    });

    if (!coupon) {
      return res.status(404).json({
        success: false,
        message: 'Coupon not found'
      });
    }

    // Check if code is being changed and if new code exists
    if (updates.code && updates.code !== coupon.code) {
      const existingCoupon = await Coupon.findOne({
        where: { 
          code: updates.code.toUpperCase(),
          id: { [Op.ne]: id }
        }
      });
      
      if (existingCoupon) {
        return res.status(400).json({
          success: false,
          message: 'Coupon code already exists'
        });
      }
      
      updates.code = updates.code.toUpperCase();
    }

    await coupon.update(updates);

    res.json({
      success: true,
      message: 'Coupon updated successfully',
      coupon
    });
  } catch (error) {
    console.error('Update coupon error:', error);
    res.status(500).json({
      success: false,
      message: 'Error updating coupon'
    });
  }
};

exports.deleteCoupon = async (req, res) => {
  try {
    const { id } = req.params;

    const coupon = await Coupon.findOne({
      where: { id, adminId: req.user.id }
    });

    if (!coupon) {
      return res.status(404).json({
        success: false,
        message: 'Coupon not found'
      });
    }

    // Check if coupon has been used
    const usageCount = await CouponUsage.count({
      where: { couponId: id }
    });

    if (usageCount > 0) {
      // Soft delete by deactivating
      await coupon.update({ isActive: false });
      res.json({
        success: true,
        message: 'Coupon deactivated successfully (has usage history)'
      });
    } else {
      // Hard delete if never used
      await coupon.destroy();
      res.json({
        success: true,
        message: 'Coupon deleted successfully'
      });
    }
  } catch (error) {
    console.error('Delete coupon error:', error);
    res.status(500).json({
      success: false,
      message: 'Error deleting coupon'
    });
  }
};

exports.getCouponUsageStats = async (req, res) => {
  try {
    const { id } = req.params;

    const coupon = await Coupon.findOne({
      where: { id, adminId: req.user.id }
    });

    if (!coupon) {
      return res.status(404).json({
        success: false,
        message: 'Coupon not found'
      });
    }

    // Get usage statistics
    const totalUsage = await CouponUsage.count({
      where: { couponId: id }
    });

    const uniqueUsers = await CouponUsage.count({
      where: { couponId: id },
      distinct: true,
      col: 'user_id'
    });

    const totalDiscount = await CouponUsage.sum('discountAmount', {
      where: { couponId: id }
    });

    // Usage over time (last 30 days)
    const thirtyDaysAgo = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000);
    const dailyUsage = await CouponUsage.findAll({
      where: {
        couponId: id,
        usedAt: { [Op.gte]: thirtyDaysAgo }
      },
      attributes: [
        [sequelize.fn('DATE', sequelize.col('used_at')), 'date'],
        [sequelize.fn('COUNT', sequelize.col('id')), 'usage_count'],
        [sequelize.fn('SUM', sequelize.col('discount_amount')), 'total_discount']
      ],
      group: [sequelize.fn('DATE', sequelize.col('used_at'))],
      order: [[sequelize.fn('DATE', sequelize.col('used_at')), 'ASC']]
    });

    res.json({
      success: true,
      stats: {
        totalUsage,
        uniqueUsers,
        totalDiscount: totalDiscount || 0,
        usageRate: coupon.usageLimit ? (totalUsage / coupon.usageLimit * 100) : null,
        remainingUses: coupon.usageLimit ? coupon.usageLimit - totalUsage : null,
        dailyUsage
      }
    });
  } catch (error) {
    console.error('Get coupon stats error:', error);
    res.status(500).json({
      success: false,
      message: 'Error fetching coupon statistics'
    });
  }
};
```

### **Step 10: Loyalty Points System**

#### **Loyalty Models & Controller**
```javascript
// models/LoyaltyPoints.js
const { DataTypes } = require('sequelize');
const sequelize = require('../config/database');

const LoyaltyPoints = sequelize.define('LoyaltyPoints', {
  userId: {
    type: DataTypes.INTEGER,
    allowNull: false,
    unique: true,
    references: { model: 'users', key: 'id' },
    field: 'user_id'
  },
  pointsEarned: {
    type: DataTypes.INTEGER,
    defaultValue: 0,
    field: 'points_earned'
  },
  pointsSpent: {
    type: DataTypes.INTEGER,
    defaultValue: 0,
    field: 'points_spent'
  },
  pointsExpired: {
    type: DataTypes.INTEGER,
    defaultValue: 0,
    field: 'points_expired'
  },
  currentBalance: {
    type: DataTypes.INTEGER,
    defaultValue: 0,
    field: 'current_balance'
  },
  tierLevel: {
    type: DataTypes.STRING(20),
    defaultValue: 'Bronze',
    field: 'tier_level'
  },
  tierProgress: {
    type: DataTypes.INTEGER,
    defaultValue: 0,
    field: 'tier_progress'
  },
  lifetimeValue: {
    type: DataTypes.DECIMAL(10, 2),
    defaultValue: 0,
    field: 'lifetime_value'
  }
}, {
  tableName: 'loyalty_points',
  timestamps: true,
  createdAt: 'created_at',
  updatedAt: 'updated_at'
});

// Instance methods
LoyaltyPoints.prototype.updateTier = function() {
  const earned = this.pointsEarned;
  
  if (earned >= 50000) {
    this.tierLevel = 'Diamond';
    this.tierProgress = Math.min(100, ((earned - 50000) / 50000) * 100);
  } else if (earned >= 25000) {
    this.tierLevel = 'Platinum';
    this.tierProgress = ((earned - 25000) / 25000) * 100;
  } else if (earned >= 10000) {
    this.tierLevel = 'Gold';
    this.tierProgress = ((earned - 10000) / 15000) * 100;
  } else if (earned >= 2500) {
    this.tierLevel = 'Silver';
    this.tierProgress = ((earned - 2500) / 7500) * 100;
  } else {
    this.tierLevel = 'Bronze';
    this.tierProgress = (earned / 2500) * 100;
  }
  
  return this;
};

LoyaltyPoints.prototype.getPointsToNextTier = function() {
  const earned = this.pointsEarned;
  
  if (earned >= 50000) return 0; // Max tier
  if (earned >= 25000) return 50000 - earned;
  if (earned >= 10000) return 25000 - earned;
  if (earned >= 2500) return 10000 - earned;
  return 2500 - earned;
};

LoyaltyPoints.prototype.getTierMultiplier = function() {
  switch (this.tierLevel) {
    case 'Diamond': return 3.0;
    case 'Platinum': return 2.5;
    case 'Gold': return 2.0;
    case 'Silver': return 1.5;
    default: return 1.0;
  }
};

module.exports = LoyaltyPoints;

// models/LoyaltyTransaction.js
const { DataTypes } = require('sequelize');
const sequelize = require('../config/database');

const LoyaltyTransaction = sequelize.define('LoyaltyTransaction', {
  userId: {
    type: DataTypes.INTEGER,
    allowNull: false,
    references: { model: 'users', key: 'id' },
    field: 'user_id'
  },
  orderId: {
    type: DataTypes.INTEGER,
    references: { model: 'orders', key: 'id' },
    field: 'order_id'
  },
  transactionType: {
    type: DataTypes.ENUM('earned', 'spent', 'expired', 'refunded', 'bonus', 'penalty'),
    allowNull: false,
    field: 'transaction_type'
  },
  points: {
    type: DataTypes.INTEGER,
    allowNull: false
  },
  balanceBefore: {
    type: DataTypes.INTEGER,
    allowNull: false,
    field: 'balance_before'
  },
  balanceAfter: {
    type: DataTypes.INTEGER,
    allowNull: false,
    field: 'balance_after'
  },
  description: DataTypes.TEXT,
  referenceId: {
    type: DataTypes.STRING(255),
    field: 'reference_id'
  },
  expiresAt: {
    type: DataTypes.DATE,
    field: 'expires_at'
  }
}, {
  tableName: 'loyalty_transactions',
  timestamps: true,
  createdAt: 'created_at',
  updatedAt: 'updated_at'
});

module.exports = LoyaltyTransaction;

// controllers/loyaltyController.js
const LoyaltyPoints = require('../models/LoyaltyPoints');
const LoyaltyTransaction = require('../models/LoyaltyTransaction');
const User = require('../models/User');
const Order = require('../models/Order');
const { Op } = require('sequelize');

exports.getLoyaltyAccount = async (req, res) => {
  try {
    const userId = req.user.id;

    let loyaltyAccount = await LoyaltyPoints.findOne({
      where: { userId },
      include: [{
        model: User,
        attributes: ['id', 'fullName', 'email']
      }]
    });

    if (!loyaltyAccount) {
      loyaltyAccount = await LoyaltyPoints.create({
        userId,
        pointsEarned: 0,
        pointsSpent: 0,
        currentBalance: 0,
        tierLevel: 'Bronze'
      });
    }

    // Update tier information
    loyaltyAccount.updateTier();
    await loyaltyAccount.save();

    // Get recent transactions
    const recentTransactions = await LoyaltyTransaction.findAll({
      where: { userId },
      order: [['createdAt', 'DESC']],
      limit: 10
    });

    // Calculate points expiring soon (next 30 days)
    const thirtyDaysFromNow = new Date(Date.now() + 30 * 24 * 60 * 60 * 1000);
    const expiringPoints = await LoyaltyTransaction.sum('points', {
      where: {
        userId,
        transactionType: 'earned',
        expiresAt: {
          [Op.lte]: thirtyDaysFromNow,
          [Op.gt]: new Date()
        }
      }
    });

    res.json({
      success: true,
      loyaltyAccount: {
        ...loyaltyAccount.toJSON(),
        pointsValue: loyaltyAccount.currentBalance * 0.01, // 1 point = $0.01
        pointsToNextTier: loyaltyAccount.getPointsToNextTier(),
        tierMultiplier: loyaltyAccount.getTierMultiplier(),
        expiringPoints: expiringPoints || 0
      },
      recentTransactions
    });
  } catch (error) {
    console.error('Get loyalty account error:', error);
    res.status(500).json({
      success: false,
      message: 'Error fetching loyalty account'
    });
  }
};

exports.getTransactionHistory = async (req, res) => {
  try {
    const userId = req.user.id;
    const { page = 1, limit = 20, type, startDate, endDate } = req.query;
    const offset = (page - 1) * limit;

    let whereClause = { userId };
    
    if (type) {
      whereClause.transactionType = type;
    }
    
    if (startDate || endDate) {
      whereClause.createdAt = {};
      if (startDate) whereClause.createdAt[Op.gte] = new Date(startDate);
      if (endDate) whereClause.createdAt[Op.lte] = new Date(endDate);
    }

    const transactions = await LoyaltyTransaction.findAndCountAll({
      where: whereClause,
      limit: parseInt(limit),
      offset: parseInt(offset),
      order: [['createdAt', 'DESC']],
      include: [{
        model: Order,
        attributes: ['id', 'orderNumber', 'totalAmount'],
        required: false
      }]
    });

    res.json({
      success: true,
      transactions: transactions.rows,
      pagination: {
        currentPage: parseInt(page),
        totalPages: Math.ceil(transactions.count / limit),
        totalItems: transactions.count,
        itemsPerPage: parseInt(limit)
      }
    });
  } catch (error) {
    console.error('Get transaction history error:', error);
    res.status(500).json({
      success: false,
      message: 'Error fetching transaction history'
    });
  }
};

exports.redeemPoints = async (req, res) => {
  try {
    const userId = req.user.id;
    const { pointsToRedeem } = req.body;

    if (!pointsToRedeem || pointsToRedeem <= 0) {
      return res.status(400).json({
        success: false,
        message: 'Invalid points amount'
      });
    }

    const loyaltyAccount = await LoyaltyPoints.findOne({
      where: { userId }
    });

    if (!loyaltyAccount || loyaltyAccount.currentBalance < pointsToRedeem) {
      return res.status(400).json({
        success: false,
        message: 'Insufficient loyalty points'
      });
    }

    const discountAmount = pointsToRedeem * 0.01; // 1 point = $0.01
    const balanceBefore = loyaltyAccount.currentBalance;

    // Update loyalty account
    await loyaltyAccount.update({
      pointsSpent: loyaltyAccount.pointsSpent + pointsToRedeem,
      currentBalance: loyaltyAccount.currentBalance - pointsToRedeem
    });

    // Create transaction record
    await LoyaltyTransaction.create({
      userId,
      transactionType: 'spent',
      points: pointsToRedeem,
      balanceBefore,
      balanceAfter: loyaltyAccount.currentBalance,
      description: `Redeemed ${pointsToRedeem} points for ${discountAmount.toFixed(2)} discount`
    });

    res.json({
      success: true,
      message: 'Points redeemed successfully',
      discountAmount,
      remainingPoints: loyaltyAccount.currentBalance,
      redemptionId: `RED-${Date.now()}-${userId}`
    });
  } catch (error) {
    console.error('Redeem points error:', error);
    res.status(500).json({
      success: false,
      message: 'Error redeeming points'
    });
  }
};

// Service function to award points (called after order completion)
exports.awardPoints = async (order) => {
  try {
    // Only B2C orders earn points
    if (order.orderType !== 'B2C') return;

    const userId = order.buyerId;
    const basePoints = Math.floor(order.totalAmount * 1); // 1 point per dollar

    let loyaltyAccount = await LoyaltyPoints.findOne({
      where: { userId }
    });

    if (!loyaltyAccount) {
      loyaltyAccount = await LoyaltyPoints.create({
        userId,
        pointsEarned: 0,
        pointsSpent: 0,
        currentBalance: 0
      });
    }

    // Apply tier multiplier
    const tierMultiplier = loyaltyAccount.getTierMultiplier();
    const pointsToAward = Math.floor(basePoints * tierMultiplier);

    if (pointsToAward <= 0) return;

    const balanceBefore = loyaltyAccount.currentBalance;

    // Update loyalty account
    await loyaltyAccount.update({
      pointsEarned: loyaltyAccount.pointsEarned + pointsToAward,
      currentBalance: loyaltyAccount.currentBalance + pointsToAward,
      lifetimeValue: loyaltyAccount.lifetimeValue + order.totalAmount
    });

    // Update tier after earning points
    loyaltyAccount.updateTier();
    await loyaltyAccount.save();

    // Create transaction record with expiration (1 year)
    const expirationDate = new Date();
    expirationDate.setFullYear(expirationDate.getFullYear() + 1);

    await LoyaltyTransaction.create({
      userId,
      orderId: order.id,
      transactionType: 'earned',
      points: pointsToAward,
      balanceBefore,
      balanceAfter: loyaltyAccount.currentBalance,
      description: `Earned ${pointsToAward} points from order #${order.orderNumber}`,
      expiresAt: expirationDate
    });

    // Send notification about points earned
    const notificationService = require('./notificationService');
    await notificationService.sendPointsEarnedNotification(userId, pointsToAward, loyaltyAccount.tierLevel);

    return pointsToAward;
  } catch (error) {
    console.error('Error awarding loyalty points:', error);
  }
};

exports.awardBonusPoints = async (req, res) => {
  try {
    const { userId, points, reason } = req.body;

    if (!userId || !points || points <= 0) {
      return res.status(400).json({
        success: false,
        message: 'Invalid request data'
      });
    }

    let loyaltyAccount = await LoyaltyPoints.findOne({
      where: { userId }
    });

    if (!loyaltyAccount) {
      loyaltyAccount = await LoyaltyPoints.create({
        userId,
        pointsEarned: 0,
        pointsSpent: 0,
        currentBalance: 0
      });
    }

    const balanceBefore = loyaltyAccount.currentBalance;

    // Update loyalty account
    await loyaltyAccount.update({
      pointsEarned: loyaltyAccount.pointsEarned + points,
      currentBalance: loyaltyAccount.currentBalance + points
    });

    // Update tier
    loyaltyAccount.updateTier();
    await loyaltyAccount.save();

    // Create transaction record
    await LoyaltyTransaction.create({
      userId,
      transactionType: 'bonus',
      points,
      balanceBefore,
      balanceAfter: loyaltyAccount.currentBalance,
      description: reason || `Bonus points awarded by admin`,
      referenceId: `BONUS-${Date.now()}-${req.user.id}`
    });

    res.json({
      success: true,
      message: 'Bonus points awarded successfully',
      pointsAwarded: points,
      newBalance: loyaltyAccount.currentBalance
    });
  } catch (error) {
    console.error('Award bonus points error:', error);
    res.status(500).json({
      success: false,
      message: 'Error awarding bonus points'
    });
  }
};

exports.getLoyaltyOverview = async (req, res) => {
  try {
    // Overall statistics
    const totalMembers = await LoyaltyPoints.count();
    const totalPointsEarned = await LoyaltyPoints.sum('pointsEarned');
    const totalPointsSpent = await LoyaltyPoints.sum('pointsSpent');
    const activeMembers = await LoyaltyPoints.count({
      where: { currentBalance: { [Op.gt]: 0 } }
    });

    // Tier distribution
    const tierDistribution = await LoyaltyPoints.findAll({
      attributes: [
        'tierLevel',
        [sequelize.fn('COUNT', sequelize.col('id')), 'count']
      ],
      group: ['tierLevel']
    });

    // Recent activity (last 30 days)
    const thirtyDaysAgo = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000);
    const recentActivity = await LoyaltyTransaction.findAll({
      where: { createdAt: { [Op.gte]: thirtyDaysAgo } },
      attributes: [
        'transactionType',
        [sequelize.fn('COUNT', sequelize.col('id')), 'count'],
        [sequelize.fn('SUM', sequelize.col('points')), 'totalPoints']
      ],
      group: ['transactionType']
    });

    // Top customers by lifetime value
    const topCustomers = await LoyaltyPoints.findAll({
      include: [{
        model: User,
        attributes: ['id', 'fullName', 'email']
      }],
      order: [['lifetimeValue', 'DESC']],
      limit: 10
    });

    res.json({
      success: true,
      overview: {
        totalMembers,
        totalPointsEarned: totalPointsEarned || 0,
        totalPointsSpent: totalPointsSpent || 0,
        totalPointsOutstanding: (totalPointsEarned || 0) - (totalPointsSpent || 0),
        activeMembers,
        membershipRate: totalMembers > 0 ? (activeMembers / totalMembers *         success: false,
        message: `Insufficient stock. Only ${cartItem.Product.stockQuantity} items available.`
      });
    }

    await cartItem.update({
      quantity,
      notes: notes !== undefined ? notes : cartItem.notes
    });

    // Update cart timestamp
    await cartItem.Cart.touch();

    res.json({
      success: true,
      message: 'Cart item updated successfully',
      cartItem
    });
  } catch (error) {
    console.error('Update cart item error:', error);
    res.status(500).json({
      success: false,
      message: 'Error updating cart item'
    });
  }
};

exports.removeCartItem = async (req, res) => {
  try {
    const { itemId } = req.params;
    const userId = req.user.id;

    const cartItem = await CartItem.findOne({
      where: { id: itemId },
      include: [{
        model: Cart,
        where: { userId }
      }]
    });

    if (!cartItem) {
      return res.status(404).json({
        success: false,
        message: 'Cart item not found'
      });
    }

    await cartItem.destroy();

    res.json({
      success: true,
      message: 'Item removed from cart successfully'
    });
  } catch (error) {
    console.error('Remove cart item error:', error);
    res.status(500).json({
      success: false,
      message: 'Error removing cart item'
    });
  }
};

exports.clearCart = async (req, res) => {
  try {
    const userId = req.user.id;
    
    const cart = await Cart.findOne({ where: { userId } });
    if (cart) {
      await CartItem.destroy({ where: { cartId: cart.id } });
      await cart.touch();
    }

    res.json({
      success: true,
      message: 'Cart cleared successfully'
    });
  } catch (error) {
    console.error('Clear cart error:', error);
    res.status(500).json({
      success: false,
      message: 'Error clearing cart'
    });
  }
};

exports.validateCart = async (req, res) => {
  try {
    const userId = req.user.id;
    const userType = req.user.role === 'supermarket' ? 'B2B' : 'B2C';

    const cart = await Cart.findOne({
      where: { userId },
      include: [{
        model: CartItem,
        include: [Product]
      }]
    });

    if (!cart || !cart.CartItems.length) {
      return res.status(400).json({
        success: false,
        message: 'Cart is empty'
      });
    }

    const issues = [];
    const validItems = [];

    for (const item of cart.CartItems) {
      if (!item.Product || !item.Product.isActive) {
        issues.push({
          itemId: item.id,
          issue: 'Product no longer available',
          action: 'remove'
        });
        continue;
      }

      if (item.Product.stockQuantity < item.quantity) {
        issues.push({
          itemId: item.id,
          productName: item.Product.name,
          requestedQuantity: item.quantity,
          availableQuantity: item.Product.stockQuantity,
          issue: 'Insufficient stock',
          action: 'reduce_quantity'
        });
        continue;
      }

      // Check price changes
      const currentPrice = item.Product.getPrice(userType);
      if (Math.abs(item.unitPrice - currentPrice) > 0.01) {
        issues.push({
          itemId: item.id,
          productName: item.Product.name,
          oldPrice: item.unitPrice,
          newPrice: currentPrice,
          issue: 'Price changed',
          action: 'update_price'
        });
      }

      validItems.push(item);
    }

    res.json({
      success: true,
      valid: issues.length === 0,
      issues,
      validItemCount: validItems.length
    });
  } catch (error) {
    console.error('Validate cart error:', error);
    res.status(500).json({
      success: false,
      message: 'Error validating cart'
    });
  }
};
```

### **Step 8: Advanced Promotion System**

#### **Promotion Models & Controller**
```javascript
// models/Promotion.js
const { DataTypes } = require('sequelize');
const sequelize = require('../config/database');

const Promotion = sequelize.define('Promotion', {
  adminId: {
    type: DataTypes.INTEGER,
    references: { model: 'users', key: 'id' },
    field: 'admin_id'
  },
  name: {
    type: DataTypes.STRING(255),
    allowNull: false
  },
  description: DataTypes.TEXT,
  promotionType: {
    type: DataTypes.ENUM('percentage', 'fixed_amount', 'buy_x_get_y', 'free_shipping', 'bundle'),
    allowNull: false,
    field: 'promotion_type'
  },
  discountValue: {
    type: DataTypes.DECIMAL(10, 2),
    field: 'discount_value'
  },
  buyQuantity: {
    type: DataTypes.INTEGER,
    field: 'buy_quantity'
  },
  getQuantity: {
    type: DataTypes.INTEGER,
    field: 'get_quantity'
  },
  targetAudience: {
    type: DataTypes.ENUM('B2B', 'B2C', 'both'),
    allowNull: false,
    field: 'target_audience'
  },
  scopeType: {
    type: DataTypes.ENUM('all_products', 'specific_products', 'category', 'minimum_order', 'brand'),
    allowNull: false,
    field: 'scope_type'
  },
  minimumOrderAmount: {
    type: DataTypes.DECIMAL(10, 2),
    field: 'minimum_order_amount'
  },
  maximumOrderAmount: {
    type: DataTypes.DECIMAL(10, 2),
    field: 'maximum_order_amount'
  },
  maxDiscountAmount: {
    type: DataTypes.DECIMAL(10, 2),
    field: 'max_discount_amount'
  },
  usageLimit: {
    type: DataTypes.INTEGER,
    field: 'usage_limit'
  },
  usageLimitPerUser: {
    type: DataTypes.INTEGER,
    defaultValue: 1,
    field: 'usage_limit_per_user'
  },
  currentUsage: {
    type: DataTypes.INTEGER,
    defaultValue: 0,
    field: 'current_usage'
  },
  startDate: {
    type: DataTypes.DATE,
    field: 'start_date'
  },
  endDate: {
    type: DataTypes.DATE,
    field: 'end_date'
  },
  isRecurring: {
    type: DataTypes.BOOLEAN,
    defaultValue: false,
    field: 'is_recurring'
  },
  recurrencePattern: {
    type: DataTypes.ENUM('daily', 'weekly', 'monthly', 'yearly', 'custom'),
    field: 'recurrence_pattern'
  },
  recurrenceDays: {
    type: DataTypes.ARRAY(DataTypes.INTEGER),
    field: 'recurrence_days'
  },
  recurrenceData: {
    type: DataTypes.JSONB,
    field: 'recurrence_data'
  },
  priority: {
    type: DataTypes.INTEGER,
    defaultValue: 0
  },
  isFeatured: {
    type: DataTypes.BOOLEAN,
    defaultValue: false,
    field: 'is_featured'
  },
  isStackable: {
    type: DataTypes.BOOLEAN,
    defaultValue: false,
    field: 'is_stackable'
  },
  requiresCoupon: {
    type: DataTypes.BOOLEAN,
    defaultValue: false,
    field: 'requires_coupon'
  },
  status: {
    type: DataTypes.ENUM('active', 'inactive', 'expired', 'scheduled', 'draft'),
    defaultValue: 'active'
  }
}, {
  tableName: 'promotions',
  timestamps: true,
  createdAt: 'created_at',
  updatedAt: 'updated_at'
});

// Instance methods
Promotion.prototype.isActive = function() {
  const now = new Date();
  return this.status === 'active' && 
         (!this.startDate || this.startDate <= now) &&
         (!this.endDate || this.endDate >= now) &&
         (!this.usageLimit || this.currentUsage < this.usageLimit);
};

Promotion.prototype.canBeUsedBy = function(userId, userUsageCount) {
  return this.isActive() && 
         (!this.usageLimitPerUser || userUsageCount < this.usageLimitPerUser);
};

module.exports = Promotion;

// controllers/promotionController.js
const Promotion = require('../models/Promotion');
const PromotionProduct = require('../models/PromotionProduct');
const PromotionCategory = require('../models/PromotionCategory');
const Product = require('../models/Product');
const Order = require('../models/Order');

exports.createPromotion = async (req, res) => {
  try {
    const {
      name, description, promotionType, discountValue, buyQuantity, getQuantity,
      targetAudience, scopeType, minimumOrderAmount, maximumOrderAmount,
      maxDiscountAmount, usageLimit, usageLimitPerUser, startDate, endDate,
      isRecurring, recurrencePattern, recurrenceDays, recurrenceData,
      priority, isFeatured, isStackable, requiresCoupon,
      productIds, categoryNames, brands
    } = req.body;

    const promotion = await Promotion.create({
      adminId: req.user.id,
      name, description, promotionType, discountValue, buyQuantity, getQuantity,
      targetAudience, scopeType, minimumOrderAmount, maximumOrderAmount,
      maxDiscountAmount, usageLimit, usageLimitPerUser, startDate, endDate,
      isRecurring, recurrencePattern, recurrenceDays, recurrenceData,
      priority, isFeatured, isStackable, requiresCoupon
    });

    // Handle scope-specific associations
    if (scopeType === 'specific_products' && productIds?.length) {
      const promotionProducts = productIds.map(productId => ({
        promotionId: promotion.id,
        productId
      }));
      await PromotionProduct.bulkCreate(promotionProducts);
    }

    if (scopeType === 'category' && categoryNames?.length) {
      const promotionCategories = categoryNames.map(categoryName => ({
        promotionId: promotion.id,
        categoryName
      }));
      await PromotionCategory.bulkCreate(promotionCategories);
    }

    res.status(201).json({
      success: true,
      message: 'Promotion created successfully',
      promotion
    });
  } catch (error) {
    console.error('Create promotion error:', error);
    res.status(500).json({
      success: false,
      message: 'Error creating promotion'
    });
  }
};

exports.getActivePromotions = async (req, res) => {
  try {
    const { userType, productId, category, orderAmount, featured } = req.query;
    const currentDate = new Date();

    let whereClause = {
      status: 'active',
      [Op.or]: [
        { startDate: { [Op.lte]: currentDate } },
        { startDate: { [Op.is]: null } }
      ],
      [Op.or]: [
        { endDate: { [Op.gte]: currentDate } },
        { endDate: { [Op.is]: null } }
      ]
    };

    if (userType) {
      whereClause.targetAudience = { [Op.in]: ['both', userType] };
    }

    if (orderAmount) {
      whereClause[Op.and] = [
        {
          [Op.or]: [
            { minimumOrderAmount: { [Op.lte]: parseFloat(orderAmount) } },
            { minimumOrderAmount: { [Op.is]: null } }
          ]
        },
        {
          [Op.or]: [
            { maximumOrderAmount: { [Op.gte]: parseFloat(orderAmount) } },
            { maximumOrderAmount: { [Op.is]: null } }
          ]
        }
      ];
    }

    if (featured === 'true') {
      whereClause.isFeatured = true;
    }

    const include = [];
    
    if (productId || scopeType === 'specific_products') {
      include.push({
        model: PromotionProduct,
        where: productId ? { productId } : {},
        required: productId ? true : false
      });
    }

    if (category || scopeType === 'category') {
      include.push({
        model: PromotionCategory,
        where: category ? { categoryName: category } : {},
        required: category ? true : false
      });
    }

    const promotions = await Promotion.findAll({
      where: whereClause,
      include,
      order: [['priority', 'DESC'], ['createdAt', 'DESC']]
    });

    res.json({
      success: true,
      promotions
    });
  } catch (error) {
    console.error('Get active promotions error:', error);
    res.status(500).json({
      success: false,
      message: 'Error fetching promotions'
    });
  }
};

exports.getFeaturedPromotions = async (req, res) => {
  try {
    const currentDate = new Date();

    const promotions = await Promotion.findAll({
      where: {
        status: 'active',
        isFeatured: true,
        [Op.or]: [
          { startDate: { [Op.lte]: currentDate } },
          { startDate: { [Op.is]: null } }
        ],
        [Op.or]: [
          { endDate: { [Op.gte]: currentDate } },
          { endDate: { [Op.is]: null } }
        ]
      },
      limit: 10,
      order: [['priority', 'DESC'], ['createdAt', 'DESC']]
    });

    res.json({
      success: true,
      promotions
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      message: 'Error fetching featured promotions'
    });
  }
};

// Recurring promotion scheduler
exports.activateRecurringPromotions = async () => {
  try {
    const currentDate = new Date();
    const currentDay = currentDate.getDay(); // 0 = Sunday, 1 = Monday, etc.
    
    // Weekly recurring promotions
    const weeklyPromotions = await Promotion.findAll({
      where: {
        isRecurring: true,
        recurrencePattern: 'weekly',
        recurrenceDays: { [Op.contains]: [currentDay] },
        status: 'scheduled'
      }
    });

    for (const promotion of weeklyPromotions) {
      await promotion.update({
        status: 'active',
        startDate: new Date(currentDate.setHours(0, 0, 0, 0)),
        endDate: new Date(currentDate.setHours(23, 59, 59, 999))
      });
      
      console.log(`Activated weekly promotion: ${promotion.name}`);
    }

    // Monthly recurring promotions
    const monthlyPromotions = await Promotion.findAll({
      where: {
        isRecurring: true,
        recurrencePattern: 'monthly',
        status: 'scheduled',
        recurrenceData: {
          dayOfMonth: currentDate.getDate()
        }
      }
    });

    for (const promotion of monthlyPromotions) {
      const endDate = new Date(currentDate);
      endDate.setDate(endDate.getDate() + (promotion.recurrenceData.duration || 1));
      
      await promotion.update({
        status: 'active',
        startDate: new Date(currentDate.setHours(0, 0, 0, 0)),
        endDate: new Date(endDate.setHours(23, 59, 59, 999))
      });
      
      console.log(`Activated monthly promotion: ${promotion.name}`);
    }

    // Special event promotions (Ramadan, holidays)
    await activateEventPromotions(currentDate);
    
  } catch (error) {
    console.error('Error activating recurring promotions:', error);
  }
};

const activateEventPromotions = async (currentDate) => {
  try {
    // Ramadan promotions
    const isRamadan = await checkIfRamadanPeriod(currentDate);
    if (isRamadan) {
      const ramadanPromotions = await Promotion.findAll({
        where: {
          recurrencePattern: 'custom',
          recurrenceData: { event: 'ramadan' },
          status: 'scheduled'
        }
      });

      for (const promotion of ramadanPromotions) {
        await promotion.update({ status: 'active' });
        console.log(`Activated Ramadan promotion: ${promotion.name}`);
      }
    }

    // Friday special promotions
    if (currentDate.getDay() === 5) { // Friday
      const fridayPromotions = await Promotion.findAll({
        where: {
          recurrencePattern: 'custom',
          recurrenceData: { event: 'friday_special' },
          status: 'scheduled'
        }
      });

      for (const promotion of fridayPromotions) {
        await promotion.update({
          status: 'active',
          startDate: new Date(currentDate.setHours(0, 0, 0, 0)),
          endDate: new Date(currentDate.setHours(23, 59, 59, 999))
        });
        console.log(`Activated Friday special: ${promotion.name}`);
      }
    }
  } catch (error) {
    console.error('Error activating event promotions:', error);
  }
};

const checkIfRamadanPeriod = async (date) => {
  // Islamic calendar integration would go here
  // For demo, using approximate dates
  const year = date.getFullYear();
  const ramadanDates = {
    2025: { start: new Date('2025-02-28'), end: new Date('2025-03-30') },
    2026: { start: new Date('2026-02-17'), end: new Date('2026-03-19') },
  };
  
  const ramadan = ramadanDates[year];
  return ramadan && date >= ramadan.start && date <= ramadan.end;
};

exports.createRamadanPromotion = async (req, res) => {
  try {
    const ramadanPromotion = await Promotion.create({
      ...req.body,
      adminId: req.user.id,
      recurrencePattern: 'custom',
      recurrenceData: { event: 'ramadan' },
      isRecurring: true,
      status: 'scheduled'
    });

    res.status(201).json({
      success: true,
      message: 'Ramadan promotion created successfully',
      promotion: ramadanPromotion
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      message: 'Error creating Ramadan promotion'
    });
  }
};

exports.createFridaySpecial = async (req, res) => {
  try {
    const fridaySpecial = await Promotion.create({
      ...req.body,
      adminId: req.user.id,
      recurrencePattern: 'custom',
      recurrenceData: { event: 'friday_special' },
      isRecurring: true,
      status: 'scheduled'
    });

    res.status(201).json({
      success: true,
      message: 'Friday special promotion created successfully',
      promotion: fridaySpecial
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      message: 'Error creating Friday special promotion'
    });
  }
};
```

### **Step 9: Coupon System**

#### **Coupon Model & Controller**
```javascript
// models/Coupon.js
const { DataTypes } = require('sequelize');
const sequelize = require('../config/database');

const Coupon = sequelize.define('Coupon', {
  adminId: {
    type: DataTypes.INTEGER,
    references: { model: 'users', key: 'id' },
    field: 'admin_id'
  },
  code: {
    type: DataTypes.STRING(50),
    allowNull: false,
    unique: true
  },
  name: DataTypes.STRING(255),
  description: DataTypes.TEXT,
  discountType: {
    type: DataTypes.ENUM('percentage', 'fixed_amount', 'free_shipping', 'buy_x_get_y'),
    allowNull: false,
    field: 'discount_type'
  },
  discountValue: {
    type: DataTypes.DECIMAL(10, 2),
    field: 'discount_value'
  },
  minimumOrderAmount: {
    type: DataTypes.DECIMAL(10, 2),
    field: 'minimum_order_amount'
  },
  maximumOrderAmount: {
    type: DataTypes.DECIMAL(10, 2),
    field: 'maximum_order_amount'
  },
  maxDiscountAmount: {
    type: DataTypes.DECIMAL(10, 2),
    field: 'max_discount_amount'
  },
  usageLimit: {
    type: DataTypes.INTEGER,
    field: 'usage_limit'
  },
  usageLimitPerUser: {
    type: DataTypes.INTEGER,
    defaultValue: 1,
    field: 'usage_limit_per_user'
  },
  currentUsage: {
    type: DataTypes.INTEGER,
    defaultValue: 0,
    field: 'current_usage'
  },
  targetAudience: {
    type: DataTypes.ENUM('B2B', 'B2C', 'both'),
    allowNull: false,
    field: 'target_audience'
  },
  applicableCategories: {
    type: DataTypes.ARRAY(DataTypes.TEXT),
    field: 'applicable_categories'
  },
  applicableProducts: {
    type: DataTypes.ARRAY(DataTypes.INTEGER),
    field: 'applicable_products'
  },
  excludedProducts: {
    type: DataTypes.ARRAY(DataTypes.INTEGER),
    field: 'excluded_products'
  },
  firstOrderOnly: {
    type: DataTypes.BOOLEAN,
    defaultValue: false,
    field: 'first_order_only'
  },
  validFrom: {
    type: DataTypes.DATE,
    field: 'valid_from'
  },
  validUntil: {
    type: DataTypes.DATE,
    field: 'valid_until'
  },
  isActive: {
    type: DataTypes.BOOLEAN,
    defaultValue: true,
    field: 'is_active'
  }
}, {
  tableName: 'coupons',
  timestamps: true,
  createdAt: 'created_at',
  updatedAt: 'updated_at'
});

// Instance methods
Coupon.prototype.isValid = function() {
  const now = new Date();
  return this.isActive &&
         (!this.validFrom || this.validFrom <= now) &&
         (!this.validUntil || this.validUntil >= now) &&
         (!this.usageLimit || this.currentUsage < this.usageLimit);
};

Coupon.prototype.canBeUsedBy = function(userId, userUsageCount, isFirstOrder = false) {
  if (!this.isValid()) return false;
  if (this.usageLimitPerUser && userUsageCount >= this.usageLimitPerUser) return false;
  if (this.firstOrderOnly && !isFirstOrder) return false;
  return true;
};

Coupon.prototype.calculateDiscount = function(orderAmount, applicableAmount = null) {
  const amount = applicableAmount || orderAmount;
  
  if (this.minimumOrderAmount && orderAmount < this.minimumOrderAmount) return 0;
  if (this.maximumOrderAmount && orderAmount > this.maximumOrderAmount) return 0;

  let discount = 0;
  
  switch (this.discountType) {
    case 'percentage':
      discount = (amount * this.discountValue) / 100;
      break;
    case 'fixed_amount':
      discount = this.discountValue;
      break;
    case 'free_shipping':
      // Handle in shipping calculation
      discount = 0;
      break;
    default:
      discount = 0;
  }

  if (this.maxDiscountAmount && discount > this.maxDiscountAmount) {
    discount = this.maxDiscountAmount;
  }

  return Math.min(discount, amount);
};

module.exports = Coupon;

// controllers/couponController.js
const Coupon = require('../models/Coupon');
const CouponUsage = require('../models/CouponUsage');
const Order = require('../models/Order');
const { Op } = require('sequelize');

exports.createCoupon = async (req, res) => {
  try {
    const {
      code, name, description, discountType, discountValue,
      minimumOrderAmount, maximumOrderAmount, maxDiscountAmount,
      usageLimit, usageLimitPerUser, targetAudience,
      applicableCategories, applicableProducts, excludedProducts,
      firstOrderOnly, validFrom, validUntil
    } = req.body;

    // Check if coupon code already exists
    const existingCoupon = await Coupon.findOne({ 
      where: { code: code.toUpperCase() } 
    });
    
    if (existingCoupon) {
      return res.status(400).json({
        success: false,
        message: 'Coupon code already exists'
      });
    }

    const coupon = await Coupon.create({
      adminId: req.user.id,
      code: code.toUpperCase(),
      name, description, discountType, discountValue,
      minimumOrderAmount, maximumOrderAmount, maxDiscountAmount,
      usageLimit, usageLimitPerUser, targetAudience,
      applicableCategories, applicableProducts, excludedProducts,
      firstOrderOnly, validFrom, validUntil
    });

    res.status(201).json({
      success: true,
      message: 'Coupon created successfully',
      coupon
    });
  } catch (error) {
    console.error('Create coupon error:', error);
    res.status(500).json({
      success: false,
      message: 'Error creating coupon'
    });
  }
};

exports.validateCoupon = async (req, res) => {
  try {
    const { code, orderAmount, cartItems = [] } = req.body;
    const userId = req.user.id;
    const userType = req.user.role === 'supermarket' ? 'B2B' : 'B2C';

    const coupon = await Coupon.findOne({
      where: { 
        code: code.toUpperCase(),
        isActive: true,
        [Op.or]: [
          { targetAudience: 'both' },
          { targetAudience: userType }
        ]
      }
    });

    if (!coupon) {
      return res.status(404).json({
        success: false,
        message: 'Invalid coupon code'
      });
    }

    if (!coupon.isValid()) {
      let message = 'Coupon is not valid';
      if (coupon.validFrom && new Date() < coupon.validFrom) {
        message = 'Coupon is not yet active';
      } else if (coupon.validUntil && new Date() > coupon.validUntil) {
        message = 'Coupon has expired';
      } else if (coupon.usageLimit && coupon.currentUsage >= coupon.usageLimit) {
        message = 'Coupon usage limit reached';
      }
      
      return res.status(400).json({
        success: false,
        message
      });
    }

    // Check per-user usage limit
    const userUsageCount = await CouponUsage.count({
      where: { couponId: coupon.id, userId }
    });

    // Check if first order only
    let isFirstOrder = false;
    if (coupon.firstOrderOnly) {
      const orderCount = await Order.count({
        where: { buyerId: userId, status: { [Op.ne]: 'cancelled' } }
      });
      isFirstOrder = orderCount === 0;
    }

    if (!coupon.canBeUsedBy(userId, userUsageCount, isFirstOrder)) {
      let message = 'You cannot use this coupon';
      if (coupon.usageLimitPerUser && userUsageCount >= coupon.usageLimitPerUser) {
        message = 'You have reached the usage limit for this coupon';
      } else if (coupon.firstOrderOnly && !isFirstOrder) {
        message = 'This coupon is only valid for first-time customers';
      }
      
      return res.status(400).json({
        success: false,
        message
      });
    }

    // Calculate applicable amount and discount
    let applicableAmount = orderAmount;
    
    if (coupon.applicableCategories?.length || coupon.applicableProducts?.length) {
      applicableAmount = 0;
      
      for (const item of cartItems) {
        let itemApplicable = false;
        
        // Check if product is specifically excluded
        if (coupon.excludedProducts?.includes(item.productId)) {
          continue;
        }
        
        // Check applicable products
        if (coupon.applicableProducts?.length) {
          itemApplicable = coupon.applicableProducts.includes(item.productId);
        }
        
        // Check applicable categories
        if (!itemApplicable && coupon.applicableCategories?.length) {
          itemApplicable = coupon.applicableCategories.includes(item.category);
        }
        
        // If no specific products/categories defined, apply to all (except excluded)
        if (!coupon.applicableProducts?.length && !coupon.applicableCategories?.length) {
          itemApplicable = true;
        }
        
        if (itemApplicable) {
          applicableAmount += item.unitPrice * item.quantity;
        }
      }
    }

    const discountAmount = coupon.calculateDiscount(orderAmount, applicableAmount);

    res.json({
      success: true,
      valid: true,
      coupon: {
        id: coupon.id,
        code: coupon.code,
        name: coupon.name,
        description: coupon.description,
        discountType: coupon.discountType,
        discountValue: coupon.discountValue,    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({
        success: false,
        message: 'Validation failed',
        errors: errors.array()
      });
    }

    const { email, password } = req.body;
    
    // Find user
    const user = await User.findOne({ where: { email } });
    if (!user) {
      return res.status(401).json({
        success: false,
        message: 'Invalid credentials'
      });
    }

    // Validate password
    const isValidPassword = await user.validatePassword(password);
    if (!isValidPassword) {
      return res.status(401).json({
        success: false,
        message: 'Invalid credentials'
      });
    }

    // Check account status
    if (user.status === 'suspended') {
      return res.status(401).json({
        success: false,
        message: 'Account has been suspended. Please contact support.'
      });
    }

    if (user.status === 'pending') {
      return res.status(401).json({
        success: false,
        message: 'Account is pending approval. Please wait for admin approval.'
      });
    }

    // Update last login
    await user.updateLastLogin();

    // Generate token
    const token = generateToken(user.id, user.role);
    
    res.json({
      success: true,
      message: 'Login successful',
      token,
      user: user.getPublicProfile()
    });
  } catch (error) {
    console.error('Login error:', error);
    res.status(500).json({
      success: false,
      message: 'Server error during login'
    });
  }
};

exports.verifyEmail = async (req, res) => {
  try {
    const { token } = req.params;
    
    // Verify token logic here
    // Update user email_verified status
    
    res.json({
      success: true,
      message: 'Email verified successfully'
    });
  } catch (error) {
    res.status(400).json({
      success: false,
      message: 'Invalid or expired verification token'
    });
  }
};

exports.forgotPassword = async (req, res) => {
  try {
    const { email } = req.body;
    
    const user = await User.findOne({ where: { email } });
    if (!user) {
      return res.status(404).json({
        success: false,
        message: 'User not found'
      });
    }

    const resetToken = generateVerificationToken();
    // Store reset token with expiration
    
    await emailService.sendPasswordResetEmail(email, resetToken);
    
    res.json({
      success: true,
      message: 'Password reset email sent'
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      message: 'Error sending password reset email'
    });
  }
};

exports.resetPassword = async (req, res) => {
  try {
    const { token, newPassword } = req.body;
    
    // Verify reset token and update password
    
    res.json({
      success: true,
      message: 'Password reset successfully'
    });
  } catch (error) {
    res.status(400).json({
      success: false,
      message: 'Invalid or expired reset token'
    });
  }
};

exports.refreshToken = async (req, res) => {
  try {
    const user = req.user; // from auth middleware
    const newToken = generateToken(user.id, user.role);
    
    res.json({
      success: true,
      token: newToken,
      user: user.getPublicProfile()
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      message: 'Error refreshing token'
    });
  }
};

exports.updateProfile = async (req, res) => {
  try {
    const userId = req.user.id;
    const { fullName, phone, companyName, address } = req.body;
    
    const user = await User.findByPk(userId);
    if (!user) {
      return res.status(404).json({
        success: false,
        message: 'User not found'
      });
    }

    await user.update({
      fullName: fullName || user.fullName,
      phone: phone || user.phone,
      companyName: companyName || user.companyName,
      address: address || user.address
    });

    res.json({
      success: true,
      message: 'Profile updated successfully',
      user: user.getPublicProfile()
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      message: 'Error updating profile'
    });
  }
};

exports.changePassword = async (req, res) => {
  try {
    const userId = req.user.id;
    const { currentPassword, newPassword } = req.body;
    
    const user = await User.findByPk(userId);
    const isValidPassword = await user.validatePassword(currentPassword);
    
    if (!isValidPassword) {
      return res.status(400).json({
        success: false,
        message: 'Current password is incorrect'
      });
    }

    await user.update({ password: newPassword });
    
    res.json({
      success: true,
      message: 'Password changed successfully'
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      message: 'Error changing password'
    });
  }
};
```

### **Step 6: Product Management System**

#### **Product Model (models/Product.js)**
```javascript
const { DataTypes } = require('sequelize');
const sequelize = require('../config/database');

const Product = sequelize.define('Product', {
  adminId: {
    type: DataTypes.INTEGER,
    allowNull: false,
    references: {
      model: 'users',
      key: 'id'
    },
    field: 'admin_id'
  },
  name: {
    type: DataTypes.STRING,
    allowNull: false,
    validate: {
      notEmpty: true,
      len: [1, 255]
    }
  },
  description: DataTypes.TEXT,
  category: {
    type: DataTypes.STRING,
    validate: { len: [1, 100] }
  },
  brand: DataTypes.STRING,
  sku: {
    type: DataTypes.STRING,
    unique: true
  },
  barcode: DataTypes.STRING,
  b2bPrice: {
    type: DataTypes.DECIMAL(10, 2),
    allowNull: false,
    validate: { min: 0 },
    field: 'b2b_price'
  },
  b2cPrice: {
    type: DataTypes.DECIMAL(10, 2),
    allowNull: false,
    validate: { min: 0 },
    field: 'b2c_price'
  },
  costPrice: {
    type: DataTypes.DECIMAL(10, 2),
    validate: { min: 0 },
    field: 'cost_price'
  },
  stockQuantity: {
    type: DataTypes.INTEGER,
    defaultValue: 0,
    validate: { min: 0 },
    field: 'stock_quantity'
  },
  minStockLevel: {
    type: DataTypes.INTEGER,
    defaultValue: 10,
    field: 'min_stock_level'
  },
  maxStockLevel: {
    type: DataTypes.INTEGER,
    defaultValue: 1000,
    field: 'max_stock_level'
  },
  unit: {
    type: DataTypes.STRING,
    defaultValue: 'piece'
  },
  weight: DataTypes.DECIMAL(8, 2),
  dimensions: DataTypes.JSONB,
  imageUrls: {
    type: DataTypes.ARRAY(DataTypes.TEXT),
    defaultValue: [],
    field: 'image_urls'
  },
  featuredImageUrl: {
    type: DataTypes.STRING,
    field: 'featured_image_url'
  },
  tags: {
    type: DataTypes.ARRAY(DataTypes.TEXT),
    defaultValue: []
  },
  isFeatured: {
    type: DataTypes.BOOLEAN,
    defaultValue: false,
    field: 'is_featured'
  },
  isActive: {
    type: DataTypes.BOOLEAN,
    defaultValue: true,
    field: 'is_active'
  },
  metaTitle: {
    type: DataTypes.STRING,
    field: 'meta_title'
  },
  metaDescription: {
    type: DataTypes.TEXT,
    field: 'meta_description'
  }
}, {
  tableName: 'products',
  timestamps: true,
  createdAt: 'created_at',
  updatedAt: 'updated_at',
  hooks: {
    beforeCreate: (product) => {
      if (!product.sku) {
        product.sku = `PRD-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`;
      }
    }
  }
});

// Instance methods
Product.prototype.updateStock = async function(quantity, type = 'set') {
  if (type === 'increment') {
    this.stockQuantity += quantity;
  } else if (type === 'decrement') {
    this.stockQuantity = Math.max(0, this.stockQuantity - quantity);
  } else {
    this.stockQuantity = quantity;
  }
  return await this.save();
};

Product.prototype.isLowStock = function() {
  return this.stockQuantity <= this.minStockLevel;
};

Product.prototype.isOutOfStock = function() {
  return this.stockQuantity === 0;
};

Product.prototype.getPrice = function(userType) {
  return userType === 'B2B' ? this.b2bPrice : this.b2cPrice;
};

Product.prototype.calculateMargin = function(userType) {
  if (!this.costPrice) return null;
  const sellingPrice = this.getPrice(userType);
  return ((sellingPrice - this.costPrice) / sellingPrice) * 100;
};

module.exports = Product;
```

#### **Product Controller (controllers/productController.js)**
```javascript
const Product = require('../models/Product');
const User = require('../models/User');
const { validationResult } = require('express-validator');
const { uploadToS3 } = require('../utils/s3Upload');
const stockService = require('../services/stockService');
const notificationService = require('../services/notificationService');

exports.createProduct = async (req, res) => {
  try {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({
        success: false,
        message: 'Validation failed',
        errors: errors.array()
      });
    }

    const {
      name, description, category, brand, b2bPrice, b2cPrice,
      costPrice, stockQuantity, minStockLevel, maxStockLevel,
      unit, weight, dimensions, tags, isFeatured
    } = req.body;

    // Handle image uploads
    let imageUrls = [];
    let featuredImageUrl = null;
    
    if (req.files && req.files.length > 0) {
      for (const file of req.files) {
        const imageUrl = await uploadToS3(file, 'products');
        imageUrls.push(imageUrl);
        if (!featuredImageUrl) featuredImageUrl = imageUrl;
      }
    }

    const product = await Product.create({
      adminId: req.user.id,
      name,
      description,
      category,
      brand,
      b2bPrice,
      b2cPrice,
      costPrice,
      stockQuantity: stockQuantity || 0,
      minStockLevel: minStockLevel || 10,
      maxStockLevel: maxStockLevel || 1000,
      unit: unit || 'piece',
      weight,
      dimensions,
      imageUrls,
      featuredImageUrl,
      tags: tags || [],
      isFeatured: isFeatured || false
    });

    // Log inventory movement
    await stockService.logInventoryMovement({
      productId: product.id,
      movementType: 'in',
      quantity: stockQuantity || 0,
      quantityBefore: 0,
      quantityAfter: stockQuantity || 0,
      referenceType: 'initial_stock',
      createdBy: req.user.id
    });

    res.status(201).json({
      success: true,
      message: 'Product created successfully',
      product
    });
  } catch (error) {
    console.error('Create product error:', error);
    res.status(500).json({
      success: false,
      message: 'Error creating product'
    });
  }
};

exports.getProducts = async (req, res) => {
  try {
    const {
      page = 1,
      limit = 20,
      category,
      brand,
      search,
      minPrice,
      maxPrice,
      userType = 'B2C',
      inStock = true,
      featured,
      sortBy = 'created_at',
      sortOrder = 'DESC'
    } = req.query;

    const offset = (page - 1) * limit;
    
    // Build where clause
    let whereClause = { isActive: true };
    
    // Role-based filtering
    if (req.user?.role === 'admin') {
      whereClause.adminId = req.user.id;
      delete whereClause.isActive; // Admin can see all their products
    }

    if (category) whereClause.category = category;
    if (brand) whereClause.brand = brand;
    if (featured !== undefined) whereClause.isFeatured = featured === 'true';
    if (inStock === 'true') whereClause.stockQuantity = { $gt: 0 };

    // Search functionality
    if (search) {
      whereClause.$or = [
        { name: { $iLike: `%${search}%` } },
        { description: { $iLike: `%${search}%` } },
        { tags: { $contains: [search] } }
      ];
    }

    // Price filtering based on user type
    const priceField = userType === 'B2B' ? 'b2bPrice' : 'b2cPrice';
    if (minPrice) whereClause[priceField] = { $gte: minPrice };
    if (maxPrice) {
      whereClause[priceField] = whereClause[priceField] 
        ? { ...whereClause[priceField], $lte: maxPrice }
        : { $lte: maxPrice };
    }

    const products = await Product.findAndCountAll({
      where: whereClause,
      limit: parseInt(limit),
      offset: parseInt(offset),
      order: [[sortBy, sortOrder.toUpperCase()]],
      include: [
        {
          model: User,
          as: 'admin',
          attributes: ['id', 'fullName', 'companyName']
        }
      ]
    });

    // Add computed fields
    const productsWithComputed = products.rows.map(product => {
      const productData = product.toJSON();
      return {
        ...productData,
        currentPrice: product.getPrice(userType),
        isLowStock: product.isLowStock(),
        isOutOfStock: product.isOutOfStock(),
        margin: product.calculateMargin(userType)
      };
    });

    res.json({
      success: true,
      products: productsWithComputed,
      pagination: {
        currentPage: parseInt(page),
        totalPages: Math.ceil(products.count / limit),
        totalItems: products.count,
        itemsPerPage: parseInt(limit)
      }
    });
  } catch (error) {
    console.error('Get products error:', error);
    res.status(500).json({
      success: false,
      message: 'Error fetching products'
    });
  }
};

exports.getProduct = async (req, res) => {
  try {
    const { id } = req.params;
    const { userType = 'B2C' } = req.query;

    let whereClause = { id };
    
    // Admin can only see their own products
    if (req.user?.role === 'admin') {
      whereClause.adminId = req.user.id;
    } else {
      whereClause.isActive = true;
    }

    const product = await Product.findOne({
      where: whereClause,
      include: [
        {
          model: User,
          as: 'admin',
          attributes: ['id', 'fullName', 'companyName']
        }
      ]
    });

    if (!product) {
      return res.status(404).json({
        success: false,
        message: 'Product not found'
      });
    }

    const productData = product.toJSON();
    
    res.json({
      success: true,
      product: {
        ...productData,
        currentPrice: product.getPrice(userType),
        isLowStock: product.isLowStock(),
        isOutOfStock: product.isOutOfStock(),
        margin: product.calculateMargin(userType)
      }
    });
  } catch (error) {
    console.error('Get product error:', error);
    res.status(500).json({
      success: false,
      message: 'Error fetching product'
    });
  }
};

exports.updateProduct = async (req, res) => {
  try {
    const { id } = req.params;
    const updates = req.body;

    const product = await Product.findOne({
      where: { id, adminId: req.user.id }
    });

    if (!product) {
      return res.status(404).json({
        success: false,
        message: 'Product not found'
      });
    }

    // Handle image uploads if present
    if (req.files && req.files.length > 0) {
      const newImageUrls = [];
      for (const file of req.files) {
        const imageUrl = await uploadToS3(file, 'products');
        newImageUrls.push(imageUrl);
      }
      updates.imageUrls = [...(product.imageUrls || []), ...newImageUrls];
      if (!product.featuredImageUrl && newImageUrls.length > 0) {
        updates.featuredImageUrl = newImageUrls[0];
      }
    }

    // Track stock changes
    if (updates.stockQuantity !== undefined && updates.stockQuantity !== product.stockQuantity) {
      const quantityBefore = product.stockQuantity;
      const quantityDiff = updates.stockQuantity - quantityBefore;
      
      await stockService.logInventoryMovement({
        productId: product.id,
        movementType: quantityDiff > 0 ? 'in' : 'adjustment',
        quantity: Math.abs(quantityDiff),
        quantityBefore,
        quantityAfter: updates.stockQuantity,
        referenceType: 'manual_adjustment',
        createdBy: req.user.id,
        notes: 'Manual stock adjustment'
      });
    }

    await product.update(updates);

    // Check for low stock alert
    if (product.isLowStock()) {
      await notificationService.sendLowStockAlert(product);
    }

    res.json({
      success: true,
      message: 'Product updated successfully',
      product
    });
  } catch (error) {
    console.error('Update product error:', error);
    res.status(500).json({
      success: false,
      message: 'Error updating product'
    });
  }
};

exports.deleteProduct = async (req, res) => {
  try {
    const { id } = req.params;

    const product = await Product.findOne({
      where: { id, adminId: req.user.id }
    });

    if (!product) {
      return res.status(404).json({
        success: false,
        message: 'Product not found'
      });
    }

    // Soft delete by setting isActive to false
    await product.update({ isActive: false });

    res.json({
      success: true,
      message: 'Product deleted successfully'
    });
  } catch (error) {
    console.error('Delete product error:', error);
    res.status(500).json({
      success: false,
      message: 'Error deleting product'
    });
  }
};

exports.bulkUpdateProducts = async (req, res) => {
  try {
    const { productIds, updates } = req.body;

    const result = await Product.update(updates, {
      where: {
        id: productIds,
        adminId: req.user.id
      }
    });

    res.json({
      success: true,
      message: `${result[0]} products updated successfully`,
      updatedCount: result[0]
    });
  } catch (error) {
    console.error('Bulk update error:', error);
    res.status(500).json({
      success: false,
      message: 'Error updating products'
    });
  }
};

exports.getCategories = async (req, res) => {
  try {
    const categories = await Product.findAll({
      attributes: [[sequelize.fn('DISTINCT', sequelize.col('category')), 'category']],
      where: { 
        category: { $ne: null },
        isActive: true
      },
      raw: true
    });

    res.json({
      success: true,
      categories: categories.map(c => c.category).filter(Boolean)
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      message: 'Error fetching categories'
    });
  }
};

exports.getBrands = async (req, res) => {
  try {
    const brands = await Product.findAll({
      attributes: [[sequelize.fn('DISTINCT', sequelize.col('brand')), 'brand']],
      where: { 
        brand: { $ne: null },
        isActive: true
      },
      raw: true
    });

    res.json({
      success: true,
      brands: brands.map(b => b.brand).filter(Boolean)
    });
  } catch (error) {
    res.status(500).json({
      success: false,
      message: 'Error fetching brands'
    });
  }
};
```

### **Step 7: Shopping Cart System**

#### **Cart Models**
```javascript
// models/Cart.js
const { DataTypes } = require('sequelize');
const sequelize = require('../config/database');

const Cart = sequelize.define('Cart', {
  userId: {
    type: DataTypes.INTEGER,
    allowNull: false,
    unique: true,
    references: {
      model: 'users',
      key: 'id'
    },
    field: 'user_id'
  },
  sessionId: {
    type: DataTypes.STRING,
    field: 'session_id'
  },
  expiresAt: {
    type: DataTypes.DATE,
    field: 'expires_at'
  }
}, {
  tableName: 'carts',
  timestamps: true,
  createdAt: 'created_at',
  updatedAt: 'updated_at'
});

module.exports = Cart;

// models/CartItem.js
const { DataTypes } = require('sequelize');
const sequelize = require('../config/database');

const CartItem = sequelize.define('CartItem', {
  cartId: {
    type: DataTypes.INTEGER,
    allowNull: false,
    references: {
      model: 'carts',
      key: 'id'
    },
    field: 'cart_id'
  },
  productId: {
    type: DataTypes.INTEGER,
    allowNull: false,
    references: {
      model: 'products',
      key: 'id'
    },
    field: 'product_id'
  },
  quantity: {
    type: DataTypes.INTEGER,
    allowNull: false,
    defaultValue: 1,
    validate: { min: 1 }
  },
  priceType: {
    type: DataTypes.ENUM('B2B', 'B2C'),
    allowNull: false,
    field: 'price_type'
  },
  unitPrice: {
    type: DataTypes.DECIMAL(10, 2),
    allowNull: false,
    field: 'unit_price'
  },
  notes: DataTypes.TEXT
}, {
  tableName: 'cart_items',
  timestamps: true,
  createdAt: 'created_at',
  updatedAt: 'updated_at'
});

module.exports = CartItem;
```

#### **Cart Controller (controllers/cartController.js)**
```javascript
const Cart = require('../models/Cart');
const CartItem = require('../models/CartItem');
const Product = require('../models/Product');
const User = require('../models/User');
const promotionService = require('../services/promotionService');
const { validationResult } = require('express-validator');

exports.getCart = async (req, res) => {
  try {
    const userId = req.user.id;
    const userType = req.user.role === 'supermarket' ? 'B2B' : 'B2C';

    let cart = await Cart.findOne({
      where: { userId },
      include: [{
        model: CartItem,
        include: [{
          model: Product,
          where: { isActive: true }
        }]
      }]
    });

    if (!cart) {
      cart = await Cart.create({ userId });
      cart.CartItems = [];
    }

    // Calculate cart totals
    let subtotal = 0;
    let totalItems = 0;
    const validItems = [];

    for (const item of cart.CartItems) {
      if (item.Product) {
        // Verify current price
        const currentPrice = item.Product.getPrice(userType);
        if (item.unitPrice !== currentPrice) {
          // Update price if changed
          await item.update({ unitPrice: currentPrice });
        }

        const itemTotal = currentPrice * item.quantity;
        subtotal += itemTotal;
        totalItems += item.quantity;
        validItems.push({
          ...item.toJSON(),
          itemTotal
        });
      }
    }

    // Apply promotions
    const promotions = await promotionService.getApplicablePromotions({
      userId,
      userType,
      cartItems: validItems,
      subtotal
    });

    let discountAmount = 0;
    let appliedPromotions = [];

    for (const promotion of promotions) {
      const discount = await promotionService.calculateDiscount(promotion, validItems, subtotal);
      if (discount > 0) {
        discountAmount += discount;
        appliedPromotions.push({
          id: promotion.id,
          name: promotion.name,
          discountAmount: discount
        });
      }
    }

    const total = subtotal - discountAmount;

    res.json({
      success: true,
      cart: {
        id: cart.id,
        items: validItems,
        subtotal,
        discountAmount,
        total,
        itemCount: totalItems,
        appliedPromotions,
        updatedAt: cart.updatedAt
      }
    });
  } catch (error) {
    console.error('Get cart error:', error);
    res.status(500).json({
      success: false,
      message: 'Error fetching cart'
    });
  }
};

exports.addToCart = async (req, res) => {
  try {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({
        success: false,
        message: 'Validation failed',
        errors: errors.array()
      });
    }

    const userId = req.user.id;
    const { productId, quantity = 1, notes } = req.body;
    const userType = req.user.role === 'supermarket' ? 'B2B' : 'B2C';

    // Get or create cart
    let cart = await Cart.findOne({ where: { userId } });
    if (!cart) {
      cart = await Cart.create({ userId });
    }

    // Get product details
    const product = await Product.findOne({
      where: { id: productId, isActive: true }
    });

    if (!product) {
      return res.status(404).json({
        success: false,
        message: 'Product not found or unavailable'
      });
    }

    // Check stock availability
    if (product.stockQuantity < quantity) {
      return res.status(400).json({
        success: false,
        message: `Insufficient stock. Only ${product.stockQuantity} items available.`
      });
    }

    // Get appropriate price
    const unitPrice = product.getPrice(userType);

    // Check if item already in cart
    let cartItem = await CartItem.findOne({
      where: { cartId: cart.id, productId }
    });

    if (cartItem) {
      // Update existing item
      const newQuantity = cartItem.quantity + quantity;
      
      if (product.stockQuantity < newQuantity) {
        return res.status(400).json({
          success: false,
          message: `Cannot add ${quantity} more items. Only ${product.stockQuantity - cartItem.quantity} additional items available.`
        });
      }

      await cartItem.update({
        quantity: newQuantity,
        unitPrice, // Update price in case it changed
        notes: notes || cartItem.notes
      });
    } else {
      // Create new cart item
      cartItem = await CartItem.create({
        cartId: cart.id,
        productId,
        quantity,
        priceType: userType,
        unitPrice,
        notes
      });
    }

    // Update cart timestamp
    await cart.touch();

    res.json({
      success: true,
      message: 'Item added to cart successfully',
      cartItem: {
        ...cartItem.toJSON(),
        product: {
          id: product.id,
          name: product.name,
          featuredImageUrl: product.featuredImageUrl
        }
      }
    });
  } catch (error) {
    console.error('Add to cart error:', error);
    res.status(500).json({
      success: false,
      message: 'Error adding item to cart'
    });
  }
};

exports.updateCartItem = async (req, res) => {
  try {
    const { itemId } = req.params;
    const { quantity, notes } = req.body;
    const userId = req.user.id;

    const cartItem = await CartItem.findOne({
      where: { id: itemId },
      include: [{
        model: Cart,
        where: { userId }
      }, {
        model: Product
      }]
    });

    if (!cartItem) {
      return res.status(404).json({
        success: false,
        message: 'Cart item not found'
      });
    }

    if (quantity <= 0) {
      await cartItem.destroy();
      return res.json({
        success: true,
        message: 'Item removed from cart'
      });
    }

    // Check stock availability
    if (cartItem.Product.stockQuantity < quantity) {
      return res.status(400).json({# TradeSuper App - Complete Development Guide
*B2B/B2C Wholesale-Retail Platform with Advanced Features*

## Table of Contents
1. [Project Overview](#project-overview)
2. [Phase 1: Backend Development](#phase-1-backend-development)
3. [Phase 2: Frontend Development](#phase-2-frontend-development)
4. [Phase 3: DevOps & Deployment](#phase-3-devops--deployment)
5. [Phase 4: Testing & Quality Assurance](#phase-4-testing--quality-assurance)
6. [Phase 5: Production & Monitoring](#phase-5-production--monitoring)

---

## Project Overview

### 🎯 **Platform Purpose**
Hybrid B2B/B2C wholesale-retail platform connecting traders (Admin) with supermarkets (B2B) and customers (B2C), managed by an App Owner who earns commissions.

### 👥 **User Roles**
- **Owner:** Platform manager (7% B2C, 3% B2B commission)
- **Admin:** Trader/Supplier (manages products, orders, promotions)
- **Supermarket:** B2B buyers (wholesale purchases)
- **Customer:** B2C buyers (retail purchases)

### 🏗️ **Architecture Stack**
- **Backend:** Node.js + Express + PostgreSQL
- **Frontend:** Flutter (iOS/Android)
- **Cloud:** AWS (S3, RDS, SNS, SES)
- **DevOps:** Docker + Kubernetes

---

# Phase 1: Backend Development

## 1.1 Environment Setup & Project Structure

### **Step 1: Initialize Backend Project**
```bash
mkdir tradesuper-backend
cd tradesuper-backend
npm init -y
```

### **Step 2: Install Core Dependencies**
```bash
# Core Framework
npm install express cors helmet morgan dotenv

# Authentication & Security
npm install jsonwebtoken bcryptjs express-rate-limit express-validator
npm install express-mongo-sanitize xss-clean hpp

# Database
npm install pg sequelize sequelize-cli

# File Upload & AWS
npm install multer aws-sdk

# Utilities
npm install nodemailer uuid winston moment
npm install node-cron lodash

# Development
npm install -D nodemon concurrently jest supertest
```

### **Step 3: Complete Project Structure**
```
tradesuper-backend/
├── config/
│   ├── database.js          # Database configuration
│   ├── aws.js               # AWS services config
│   ├── redis.js             # Redis cache config
│   └── scheduler.js         # Cron job configuration
├── controllers/
│   ├── authController.js    # Authentication logic
│   ├── userController.js    # User management
│   ├── productController.js # Product CRUD operations
│   ├── cartController.js    # Shopping cart logic
│   ├── orderController.js   # Order processing
│   ├── promotionController.js # Promotion management
│   ├── couponController.js  # Coupon system
│   ├── loyaltyController.js # Loyalty points system
│   ├── notificationController.js # Notification handling
│   ├── reorderController.js # Reorder templates
│   ├── wishlistController.js # Wishlist management
│   ├── reviewController.js  # Product reviews
│   └── analyticsController.js # Business analytics
├── middleware/
│   ├── auth.js              # JWT authentication
│   ├── roleCheck.js         # Role-based access control
│   ├── upload.js            # File upload handling
│   ├── validation.js        # Input validation
│   ├── rateLimiter.js       # API rate limiting
│   └── errorHandler.js      # Global error handling
├── models/
│   ├── User.js              # User model
│   ├── Product.js           # Product model
│   ├── Cart.js              # Shopping cart model
│   ├── CartItem.js          # Cart items model
│   ├── Order.js             # Order model
│   ├── OrderItem.js         # Order items model
│   ├── Promotion.js         # Promotion model
│   ├── PromotionProduct.js  # Promotion-product relations
│   ├── PromotionCategory.js # Promotion-category relations
│   ├── Coupon.js            # Coupon model
│   ├── CouponUsage.js       # Coupon usage tracking
│   ├── LoyaltyPoints.js     # Loyalty account model
│   ├── LoyaltyTransaction.js # Loyalty transaction history
│   ├── Notification.js      # Notification model
│   ├── NotificationPreference.js # User notification settings
│   ├── ReorderTemplate.js   # Reorder template model
│   ├── ReorderTemplateItem.js # Reorder template items
│   ├── Wishlist.js          # Wishlist model
│   ├── ProductReview.js     # Product review model
│   ├── Commission.js        # Commission tracking
│   └── index.js             # Model associations
├── routes/
│   ├── auth.js              # Authentication routes
│   ├── users.js             # User management routes
│   ├── products.js          # Product routes
│   ├── cart.js              # Cart routes
│   ├── orders.js            # Order routes
│   ├── promotions.js        # Promotion routes
│   ├── coupons.js           # Coupon routes
│   ├── loyalty.js           # Loyalty routes
│   ├── notifications.js     # Notification routes
│   ├── reorder.js           # Reorder routes
│   ├── wishlist.js          # Wishlist routes
│   ├── reviews.js           # Review routes
│   └── analytics.js         # Analytics routes
├── services/
│   ├── authService.js       # Authentication business logic
│   ├── emailService.js      # Email notification service
│   ├── pushNotificationService.js # Push notification service
│   ├── smsService.js        # SMS service
│   ├── orderService.js      # Order processing service
│   ├── loyaltyService.js    # Loyalty points service
│   ├── promotionService.js  # Promotion calculation service
│   ├── analyticsService.js  # Analytics calculation service
│   ├── paymentService.js    # Payment processing service
│   └── stockService.js      # Inventory management service
├── jobs/
│   ├── cronJobs.js          # Scheduled jobs setup
│   ├── promotionScheduler.js # Promotion automation
│   ├── stockAlerts.js       # Low stock notifications
│   ├── loyaltyJobs.js       # Loyalty point calculations
│   └── analyticsJobs.js     # Daily analytics processing
├── utils/
│   ├── helpers.js           # Utility functions
│   ├── validators.js        # Input validation rules
│   ├── constants.js         # Application constants
│   ├── priceCalculator.js   # Price calculation utilities
│   ├── dateUtils.js         # Date manipulation utilities
│   ├── emailTemplates.js    # Email HTML templates
│   └── responseFormatter.js # API response formatting
├── migrations/             # Database migrations
├── seeders/               # Database seeders
├── tests/                 # Test files
├── logs/                  # Application logs
├── uploads/               # Temporary file uploads
├── server.js              # Main application entry
├── package.json
├── .env.example
├── .gitignore
└── README.md
```

## 1.2 Database Design & Models

### **Step 4: Database Schema Setup**

#### **Enhanced Database Tables**
```sql
-- Users table (Owner, Admin, Supermarket, Customer)
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    full_name VARCHAR(255) NOT NULL,
    phone VARCHAR(20),
    role VARCHAR(20) CHECK (role IN ('owner', 'admin', 'supermarket', 'customer')),
    status VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'suspended', 'pending')),
    email_verified BOOLEAN DEFAULT FALSE,
    phone_verified BOOLEAN DEFAULT FALSE,
    last_login TIMESTAMP,
    profile_image_url VARCHAR(500),
    company_name VARCHAR(255), -- for supermarkets
    tax_id VARCHAR(100), -- for business accounts
    address JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Products table
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    admin_id INTEGER REFERENCES users(id),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    category VARCHAR(100),
    brand VARCHAR(100),
    sku VARCHAR(100) UNIQUE,
    barcode VARCHAR(100),
    b2b_price DECIMAL(10,2) NOT NULL,
    b2c_price DECIMAL(10,2) NOT NULL,
    cost_price DECIMAL(10,2),
    stock_quantity INTEGER DEFAULT 0,
    min_stock_level INTEGER DEFAULT 10,
    max_stock_level INTEGER DEFAULT 1000,
    unit VARCHAR(50) DEFAULT 'piece',
    weight DECIMAL(8,2),
    dimensions JSONB, -- {length, width, height}
    image_urls TEXT[],
    featured_image_url VARCHAR(500),
    tags TEXT[],
    is_featured BOOLEAN DEFAULT FALSE,
    is_active BOOLEAN DEFAULT TRUE,
    meta_title VARCHAR(255),
    meta_description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Shopping carts
CREATE TABLE carts (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) UNIQUE,
    session_id VARCHAR(255), -- for guest carts
    expires_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Cart items
CREATE TABLE cart_items (
    id SERIAL PRIMARY KEY,
    cart_id INTEGER REFERENCES carts(id) ON DELETE CASCADE,
    product_id INTEGER REFERENCES products(id),
    quantity INTEGER NOT NULL DEFAULT 1,
    price_type VARCHAR(10) CHECK (price_type IN ('B2B', 'B2C')),
    unit_price DECIMAL(10,2) NOT NULL,
    notes TEXT, -- special requests
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Enhanced Orders table
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    order_number VARCHAR(50) UNIQUE NOT NULL,
    buyer_id INTEGER REFERENCES users(id),
    admin_id INTEGER REFERENCES users(id),
    order_type VARCHAR(10) CHECK (order_type IN ('B2B', 'B2C')),
    subtotal DECIMAL(10,2) NOT NULL,
    discount_amount DECIMAL(10,2) DEFAULT 0,
    coupon_discount DECIMAL(10,2) DEFAULT 0,
    loyalty_points_used INTEGER DEFAULT 0,
    loyalty_discount DECIMAL(10,2) DEFAULT 0,
    shipping_cost DECIMAL(10,2) DEFAULT 0,
    tax_amount DECIMAL(10,2) DEFAULT 0,
    total_amount DECIMAL(10,2) NOT NULL,
    commission_rate DECIMAL(5,4) NOT NULL,
    commission_amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending', 'confirmed', 'processing', 'shipped', 'delivered', 'cancelled', 'refunded')),
    payment_status VARCHAR(20) DEFAULT 'pending' CHECK (payment_status IN ('pending', 'paid', 'failed', 'refunded', 'partial')),
    shipping_address JSONB,
    billing_address JSONB,
    payment_method VARCHAR(50),
    payment_reference VARCHAR(255),
    promotion_id INTEGER REFERENCES promotions(id),
    coupon_id INTEGER REFERENCES coupons(id),
    delivery_date DATE,
    delivery_time_slot VARCHAR(50),
    tracking_number VARCHAR(100),
    notes TEXT,
    internal_notes TEXT, -- admin notes
    cancelled_reason TEXT,
    refund_amount DECIMAL(10,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Order items
CREATE TABLE order_items (
    id SERIAL PRIMARY KEY,
    order_id INTEGER REFERENCES orders(id) ON DELETE CASCADE,
    product_id INTEGER REFERENCES products(id),
    product_name VARCHAR(255) NOT NULL, -- snapshot
    product_sku VARCHAR(100),
    quantity INTEGER NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    total_price DECIMAL(10,2) NOT NULL,
    discount_amount DECIMAL(10,2) DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Enhanced Promotions table
CREATE TABLE promotions (
    id SERIAL PRIMARY KEY,
    admin_id INTEGER REFERENCES users(id),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    promotion_type VARCHAR(20) CHECK (promotion_type IN ('percentage', 'fixed_amount', 'buy_x_get_y', 'free_shipping', 'bundle')),
    discount_value DECIMAL(10,2),
    buy_quantity INTEGER, -- for buy_x_get_y
    get_quantity INTEGER, -- for buy_x_get_y
    target_audience VARCHAR(20) CHECK (target_audience IN ('B2B', 'B2C', 'both')),
    scope_type VARCHAR(20) CHECK (scope_type IN ('all_products', 'specific_products', 'category', 'minimum_order', 'brand')),
    minimum_order_amount DECIMAL(10,2),
    maximum_order_amount DECIMAL(10,2),
    max_discount_amount DECIMAL(10,2),
    usage_limit INTEGER,
    usage_limit_per_user INTEGER DEFAULT 1,
    current_usage INTEGER DEFAULT 0,
    start_date TIMESTAMP,
    end_date TIMESTAMP,
    is_recurring BOOLEAN DEFAULT FALSE,
    recurrence_pattern VARCHAR(20) CHECK (recurrence_pattern IN ('daily', 'weekly', 'monthly', 'yearly', 'custom')),
    recurrence_days INTEGER[], -- [1,2,3,4,5,6,7] for days of week
    recurrence_data JSONB, -- custom recurrence rules
    priority INTEGER DEFAULT 0, -- for overlapping promotions
    is_featured BOOLEAN DEFAULT FALSE,
    is_stackable BOOLEAN DEFAULT FALSE,
    requires_coupon BOOLEAN DEFAULT FALSE,
    status VARCHAR(20) DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'expired', 'scheduled', 'draft')),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Promotion products (for specific product promotions)
CREATE TABLE promotion_products (
    id SERIAL PRIMARY KEY,
    promotion_id INTEGER REFERENCES promotions(id) ON DELETE CASCADE,
    product_id INTEGER REFERENCES products(id) ON DELETE CASCADE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Promotion categories
CREATE TABLE promotion_categories (
    id SERIAL PRIMARY KEY,
    promotion_id INTEGER REFERENCES promotions(id) ON DELETE CASCADE,
    category_name VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Coupons table
CREATE TABLE coupons (
    id SERIAL PRIMARY KEY,
    admin_id INTEGER REFERENCES users(id),
    code VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(255),
    description TEXT,
    discount_type VARCHAR(20) CHECK (discount_type IN ('percentage', 'fixed_amount', 'free_shipping', 'buy_x_get_y')),
    discount_value DECIMAL(10,2),
    minimum_order_amount DECIMAL(10,2),
    maximum_order_amount DECIMAL(10,2),
    max_discount_amount DECIMAL(10,2),
    usage_limit INTEGER,
    usage_limit_per_user INTEGER DEFAULT 1,
    current_usage INTEGER DEFAULT 0,
    target_audience VARCHAR(20) CHECK (target_audience IN ('B2B', 'B2C', 'both')),
    applicable_categories TEXT[],
    applicable_products INTEGER[],
    excluded_products INTEGER[],
    first_order_only BOOLEAN DEFAULT FALSE,
    valid_from TIMESTAMP,
    valid_until TIMESTAMP,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Coupon usage tracking
CREATE TABLE coupon_usage (
    id SERIAL PRIMARY KEY,
    coupon_id INTEGER REFERENCES coupons(id),
    user_id INTEGER REFERENCES users(id),
    order_id INTEGER REFERENCES orders(id),
    discount_amount DECIMAL(10,2),
    used_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Loyalty points system
CREATE TABLE loyalty_points (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) UNIQUE,
    points_earned INTEGER DEFAULT 0,
    points_spent INTEGER DEFAULT 0,
    points_expired INTEGER DEFAULT 0,
    current_balance INTEGER DEFAULT 0,
    tier_level VARCHAR(20) DEFAULT 'Bronze',
    tier_progress INTEGER DEFAULT 0,
    lifetime_value DECIMAL(10,2) DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Loyalty point transactions
CREATE TABLE loyalty_transactions (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    order_id INTEGER REFERENCES orders(id),
    transaction_type VARCHAR(20) CHECK (transaction_type IN ('earned', 'spent', 'expired', 'refunded', 'bonus', 'penalty')),
    points INTEGER NOT NULL,
    balance_before INTEGER NOT NULL,
    balance_after INTEGER NOT NULL,
    description TEXT,
    reference_id VARCHAR(255),
    expires_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Notifications system
CREATE TABLE notifications (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    title VARCHAR(255) NOT NULL,
    message TEXT NOT NULL,
    notification_type VARCHAR(30) CHECK (notification_type IN ('order_update', 'promotion', 'stock_alert', 'system', 'loyalty_points', 'coupon', 'payment', 'review')),
    priority VARCHAR(10) CHECK (priority IN ('low', 'medium', 'high', 'urgent')) DEFAULT 'medium',
    related_id INTEGER,
    related_type VARCHAR(50),
    action_url VARCHAR(500),
    image_url VARCHAR(500),
    is_read BOOLEAN DEFAULT FALSE,
    is_sent BOOLEAN DEFAULT FALSE,
    send_push BOOLEAN DEFAULT TRUE,
    send_email BOOLEAN DEFAULT FALSE,
    send_sms BOOLEAN DEFAULT FALSE,
    scheduled_for TIMESTAMP,
    sent_at TIMESTAMP,
    read_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- User notification preferences
CREATE TABLE notification_preferences (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id) UNIQUE,
    push_notifications BOOLEAN DEFAULT TRUE,
    email_notifications BOOLEAN DEFAULT TRUE,
    sms_notifications BOOLEAN DEFAULT FALSE,
    marketing_emails BOOLEAN DEFAULT TRUE,
    order_notifications BOOLEAN DEFAULT TRUE,
    promotion_notifications BOOLEAN DEFAULT TRUE,
    stock_notifications BOOLEAN DEFAULT FALSE,
    review_notifications BOOLEAN DEFAULT TRUE,
    quiet_hours_start TIME,
    quiet_hours_end TIME,
    timezone VARCHAR(50) DEFAULT 'UTC',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Reorder functionality
CREATE TABLE reorder_templates (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    is_favorite BOOLEAN DEFAULT FALSE,
    is_auto_reorder BOOLEAN DEFAULT FALSE,
    auto_reorder_frequency VARCHAR(20),
    last_ordered_at TIMESTAMP,
    next_auto_order TIMESTAMP,
    total_amount DECIMAL(10,2),
    item_count INTEGER DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Reorder template items
CREATE TABLE reorder_template_items (
    id SERIAL PRIMARY KEY,
    template_id INTEGER REFERENCES reorder_templates(id) ON DELETE CASCADE,
    product_id INTEGER REFERENCES products(id),
    product_name VARCHAR(255) NOT NULL, -- snapshot
    quantity INTEGER NOT NULL,
    price_type VARCHAR(10) CHECK (price_type IN ('B2B', 'B2C')),
    last_unit_price DECIMAL(10,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Wishlist/Favorites
CREATE TABLE wishlists (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    product_id INTEGER REFERENCES products(id),
    notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, product_id)
);

-- Product reviews and ratings
CREATE TABLE product_reviews (
    id SERIAL PRIMARY KEY,
    product_id INTEGER REFERENCES products(id),
    user_id INTEGER REFERENCES users(id),
    order_id INTEGER REFERENCES orders(id),
    rating INTEGER CHECK (rating >= 1 AND rating <= 5),
    title VARCHAR(255),
    review_text TEXT,
    pros TEXT,
    cons TEXT,
    images TEXT[],
    is_verified_purchase BOOLEAN DEFAULT FALSE,
    is_approved BOOLEAN DEFAULT TRUE,
    helpful_count INTEGER DEFAULT 0,
    reported_count INTEGER DEFAULT 0,
    admin_response TEXT,
    admin_response_date TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(product_id, user_id, order_id)
);

-- Commission tracking
CREATE TABLE commissions (
    id SERIAL PRIMARY KEY,
    order_id INTEGER REFERENCES orders(id),
    owner_id INTEGER REFERENCES users(id),
    admin_id INTEGER REFERENCES users(id),
    commission_amount DECIMAL(10,2) NOT NULL,
    commission_rate DECIMAL(5,4) NOT NULL,
    order_type VARCHAR(10) CHECK (order_type IN ('B2B', 'B2C')),
    status VARCHAR(20) DEFAULT 'pending' CHECK (status IN ('pending', 'paid', 'cancelled')),
    paid_at TIMESTAMP,
    payment_reference VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Analytics and reporting
CREATE TABLE analytics_daily (
    id SERIAL PRIMARY KEY,
    date DATE NOT NULL,
    admin_id INTEGER REFERENCES users(id),
    total_orders INTEGER DEFAULT 0,
    b2b_orders INTEGER DEFAULT 0,
    b2c_orders INTEGER DEFAULT 0,
    total_revenue DECIMAL(12,2) DEFAULT 0,
    b2b_revenue DECIMAL(12,2) DEFAULT 0,
    b2c_revenue DECIMAL(12,2) DEFAULT 0,
    total_commission DECIMAL(10,2) DEFAULT 0,
    new_customers INTEGER DEFAULT 0,
    returning_customers INTEGER DEFAULT 0,
    products_sold INTEGER DEFAULT 0,
    avg_order_value DECIMAL(10,2) DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(date, admin_id)
);

-- Product categories
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    parent_id INTEGER REFERENCES categories(id),
    slug VARCHAR(255) UNIQUE,
    image_url VARCHAR(500),
    is_active BOOLEAN DEFAULT TRUE,
    sort_order INTEGER DEFAULT 0,
    meta_title VARCHAR(255),
    meta_description TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Inventory tracking
CREATE TABLE inventory_movements (
    id SERIAL PRIMARY KEY,
    product_id INTEGER REFERENCES products(id),
    movement_type VARCHAR(20) CHECK (movement_type IN ('in', 'out', 'adjustment', 'damaged', 'expired')),
    quantity INTEGER NOT NULL,
    quantity_before INTEGER NOT NULL,
    quantity_after INTEGER NOT NULL,
    reference_type VARCHAR(50), -- 'order', 'purchase', 'adjustment'
    reference_id INTEGER,
    unit_cost DECIMAL(10,2),
    total_cost DECIMAL(10,2),
    notes TEXT,
    created_by INTEGER REFERENCES users(id),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Device tokens for push notifications
CREATE TABLE device_tokens (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),
    device_id VARCHAR(255) NOT NULL,
    token VARCHAR(500) NOT NULL,
    platform VARCHAR(20) CHECK (platform IN ('ios', 'android', 'web')),
    app_version VARCHAR(50),
    is_active BOOLEAN DEFAULT TRUE,
    last_used TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(user_id, device_id)
);

-- Create indexes for performance
CREATE INDEX idx_products_admin_id ON products(admin_id);
CREATE INDEX idx_products_category ON products(category);
CREATE INDEX idx_products_status ON products(is_active);
CREATE INDEX idx_orders_buyer_id ON orders(buyer_id);
CREATE INDEX idx_orders_admin_id ON orders(admin_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_date ON orders(created_at);
CREATE INDEX idx_notifications_user_id ON notifications(user_id);
CREATE INDEX idx_notifications_unread ON notifications(user_id, is_read);
CREATE INDEX idx_loyalty_transactions_user_id ON loyalty_transactions(user_id);
CREATE INDEX idx_analytics_date_admin ON analytics_daily(date, admin_id);
```

## 1.3 Core Backend Features Implementation

### **Step 5: Authentication & User Management**

#### **Enhanced User Model (models/User.js)**
```javascript
const { DataTypes } = require('sequelize');
const sequelize = require('../config/database');
const bcrypt = require('bcryptjs');

const User = sequelize.define('User', {
  email: {
    type: DataTypes.STRING,
    allowNull: false,
    unique: true,
    validate: { isEmail: true }
  },
  password: {
    type: DataTypes.STRING,
    allowNull: false,
    validate: { len: [6, 255] }
  },
  fullName: {
    type: DataTypes.STRING,
    allowNull: false,
    field: 'full_name'
  },
  phone: {
    type: DataTypes.STRING,
    validate: { len: [10, 20] }
  },
  role: {
    type: DataTypes.ENUM('owner', 'admin', 'supermarket', 'customer'),
    allowNull: false
  },
  status: {
    type: DataTypes.ENUM('active', 'suspended', 'pending'),
    defaultValue: 'active'
  },
  emailVerified: {
    type: DataTypes.BOOLEAN,
    defaultValue: false,
    field: 'email_verified'
  },
  phoneVerified: {
    type: DataTypes.BOOLEAN,
    defaultValue: false,
    field: 'phone_verified'
  },
  lastLogin: {
    type: DataTypes.DATE,
    field: 'last_login'
  },
  profileImageUrl: {
    type: DataTypes.STRING,
    field: 'profile_image_url'
  },
  companyName: {
    type: DataTypes.STRING,
    field: 'company_name'
  },
  taxId: {
    type: DataTypes.STRING,
    field: 'tax_id'
  },
  address: {
    type: DataTypes.JSONB
  }
}, {
  tableName: 'users',
  timestamps: true,
  createdAt: 'created_at',
  updatedAt: 'updated_at',
  hooks: {
    beforeCreate: async (user) => {
      if (user.password) {
        user.password = await bcrypt.hash(user.password, 12);
      }
    },
    beforeUpdate: async (user) => {
      if (user.changed('password')) {
        user.password = await bcrypt.hash(user.password, 12);
      }
    }
  }
});

User.prototype.validatePassword = async function(password) {
  return await bcrypt.compare(password, this.password);
};

User.prototype.updateLastLogin = async function() {
  this.lastLogin = new Date();
  return await this.save();
};

User.prototype.getPublicProfile = function() {
  const { password, ...publicData } = this.toJSON();
  return publicData;
};

module.exports = User;
```

#### **Authentication Controller (controllers/authController.js)**
```javascript
const jwt = require('jsonwebtoken');
const crypto = require('crypto');
const User = require('../models/User');
const emailService = require('../services/emailService');
const { validationResult } = require('express-validator');

const generateToken = (userId, role) => {
  return jwt.sign(
    { userId, role },
    process.env.JWT_SECRET,
    { expiresIn: process.env.JWT_EXPIRES_IN || '7d' }
  );
};

const generateVerificationToken = () => {
  return crypto.randomBytes(32).toString('hex');
};

exports.register = async (req, res) => {
  try {
    // Check validation errors
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({
        success: false,
        message: 'Validation failed',
        errors: errors.array()
      });
    }

    const { email, password, fullName, phone, role, companyName, taxId } = req.body;
    
    // Check if user already exists
    const existingUser = await User.findOne({ where: { email } });
    if (existingUser) {
      return res.status(400).json({
        success: false,
        message: 'User with this email already exists'
      });
    }

    // Create user
    const userData = {
      email,
      password,
      fullName,
      phone,
      role: role || 'customer',
      companyName,
      taxId
    };

    // Business accounts need approval
    if (['admin', 'supermarket'].includes(userData.role)) {
      userData.status = 'pending';
    }

    const user = await User.create(userData);

    // Generate verification token if email verification is required
    let verificationToken = null;
    if (process.env.REQUIRE_EMAIL_VERIFICATION === 'true') {
      verificationToken = generateVerificationToken();
      // Store verification token in cache/database
      await emailService.sendVerificationEmail(user.email, verificationToken);
    }

    // Generate JWT token
    const token = generateToken(user.id, user.role);
    
    res.status(201).json({
      success: true,
      message: 'User created successfully',
      token,
      user: user.getPublicProfile(),
      requiresVerification: !!verificationToken
    });
  } catch (error) {
    console.error('Registration error:', error);
    res.status(500).json({
      success: false,
      message: 'Server error during registration',
      error: process.env.NODE_ENV === 'development' ? error.message : undefined
    });
  }
};

exports.login = async (req, res) => {
  try {