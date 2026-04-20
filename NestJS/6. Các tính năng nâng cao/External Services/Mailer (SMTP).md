# Gửi Email trong NestJS với SMTP (Dùng pnpm)

## 📦 Cài đặt với pnpm

```bash
# Cài đặt package chính
pnpm add @nestjs-modules/mailer nodemailer

# Cài đặt template engine (tuỳ chọn)
pnpm add handlebars

# Cài đặt types cho TypeScript
pnpm add -D @types/nodemailer
```

## 🏗 Cấu trúc thư mục

```
src/
├── modules/
│   └── mail/
│       ├── mail.module.ts
│       ├── mail.service.ts
│       ├── mail.controller.ts (tuỳ chọn)
│       └── templates/
│           ├── welcome.hbs
│           ├── reset-password.hbs
│           └── otp.hbs
├── config/
│   └── mail.config.ts
└── .env
```

## 🔧 Cấu hình chi tiết

### 1. **Environment Variables (.env)**

```env
# Email Configuration
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USER=your-email@gmail.com
MAIL_PASSWORD=your-app-password
MAIL_FROM=noreply@yourapp.com
MAIL_FROM_NAME="Your App Name"

# For production
MAIL_SECURE=false
MAIL_DEBUG=false
```

### 2. **Mail Configuration File**

```typescript
// config/mail.config.ts
import { MailerOptions } from '@nestjs-modules/mailer';
import { HandlebarsAdapter } from '@nestjs-modules/mailer/dist/adapters/handlebars.adapter';
import { join } from 'path';

export const mailConfig = (): MailerOptions => ({
  transport: {
    host: process.env.MAIL_HOST,
    port: parseInt(process.env.MAIL_PORT || '587'),
    secure: process.env.MAIL_SECURE === 'true', // true for 465, false for other ports
    auth: {
      user: process.env.MAIL_USER,
      pass: process.env.MAIL_PASSWORD,
    },
    tls: {
      rejectUnauthorized: false, // chỉ dùng cho development
    },
  },
  defaults: {
    from: `"${process.env.MAIL_FROM_NAME}" <${process.env.MAIL_FROM}>`,
  },
  template: {
    dir: join(__dirname, '..', 'modules', 'mail', 'templates'),
    adapter: new HandlebarsAdapter(),
    options: {
      strict: true,
    },
  },
  options: {
    partials: {
      dir: join(__dirname, '..', 'modules', 'mail', 'templates', 'partials'),
    },
  },
});
```

### 3. **Mail Module**

```typescript
// modules/mail/mail.module.ts
import { Module } from '@nestjs/common';
import { MailerModule } from '@nestjs-modules/mailer';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { MailService } from './mail.service';
import { MailController } from './mail.controller';
import { HandlebarsAdapter } from '@nestjs-modules/mailer/dist/adapters/handlebars.adapter';
import { join } from 'path';

@Module({
  imports: [
    MailerModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: async (configService: ConfigService) => ({
        transport: {
          host: configService.get<string>('MAIL_HOST'),
          port: configService.get<number>('MAIL_PORT'),
          secure: configService.get<string>('MAIL_SECURE') === 'true',
          auth: {
            user: configService.get<string>('MAIL_USER'),
            pass: configService.get<string>('MAIL_PASSWORD'),
          },
        },
        defaults: {
          from: `"${configService.get<string>('MAIL_FROM_NAME')}" <${configService.get<string>('MAIL_FROM')}>`,
        },
        template: {
          dir: join(__dirname, 'templates'),
          adapter: new HandlebarsAdapter(),
          options: {
            strict: true,
          },
        },
      }),
    }),
  ],
  controllers: [MailController],
  providers: [MailService],
  exports: [MailService],
})
export class MailModule {}
```

### 4. **Mail Service**

```typescript
// modules/mail/mail.service.ts
import { Injectable, Logger } from '@nestjs/common';
import { MailerService } from '@nestjs-modules/mailer';
import { ConfigService } from '@nestjs/config';

export interface MailOptions {
  to: string | string[];
  subject: string;
  template?: string;
  context?: Record<string, any>;
  html?: string;
  text?: string;
  attachments?: Array<{
    filename: string;
    content?: string | Buffer;
    path?: string;
    contentType?: string;
  }>;
}

@Injectable()
export class MailService {
  private readonly logger = new Logger(MailService.name);

  constructor(
    private readonly mailerService: MailerService,
    private readonly configService: ConfigService,
  ) {}

  /**
   * Gửi email sử dụng template
   */
  async sendTemplateEmail(options: MailOptions): Promise<void> {
    try {
      const { to, subject, template, context, attachments } = options;

      await this.mailerService.sendMail({
        to,
        subject,
        template,
        context,
        attachments,
      });

      this.logger.log(`Email sent successfully to ${to}`);
    } catch (error) {
      this.logger.error(`Failed to send email to ${options.to}:`, error.stack);
      throw new Error(`Email sending failed: ${error.message}`);
    }
  }

  /**
   * Gửi email HTML thuần
   */
  async sendHtmlEmail(to: string, subject: string, html: string): Promise<void> {
    try {
      await this.mailerService.sendMail({
        to,
        subject,
        html,
      });
      this.logger.log(`HTML email sent to ${to}`);
    } catch (error) {
      this.logger.error(`Failed to send HTML email to ${to}:`, error.stack);
      throw error;
    }
  }

  /**
   * Gửi email text thuần
   */
  async sendTextEmail(to: string, subject: string, text: string): Promise<void> {
    try {
      await this.mailerService.sendMail({
        to,
        subject,
        text,
      });
      this.logger.log(`Text email sent to ${to}`);
    } catch (error) {
      this.logger.error(`Failed to send text email to ${to}:`, error.stack);
      throw error;
    }
  }

  /**
   * Gửi email OTP
   */
  async sendOtpEmail(email: string, otp: string, name?: string): Promise<void> {
    await this.sendTemplateEmail({
      to: email,
      subject: 'Mã OTP xác thực tài khoản',
      template: 'otp',
      context: {
        name: name || 'User',
        otp: otp,
        expiryMinutes: 5,
        year: new Date().getFullYear(),
      },
    });
  }

  /**
   * Gửi email welcome
   */
  async sendWelcomeEmail(email: string, name: string): Promise<void> {
    await this.sendTemplateEmail({
      to: email,
      subject: 'Chào mừng bạn đến với chúng tôi!',
      template: 'welcome',
      context: {
        name: name,
        loginUrl: `${this.configService.get('APP_URL')}/login`,
        year: new Date().getFullYear(),
      },
    });
  }

  /**
   * Gửi email reset password
   */
  async sendResetPasswordEmail(email: string, resetToken: string, name?: string): Promise<void> {
    const resetUrl = `${this.configService.get('APP_URL')}/reset-password?token=${resetToken}`;
    
    await this.sendTemplateEmail({
      to: email,
      subject: 'Yêu cầu đặt lại mật khẩu',
      template: 'reset-password',
      context: {
        name: name || 'User',
        resetUrl: resetUrl,
        expiryHours: 24,
        year: new Date().getFullYear(),
      },
    });
  }

  /**
   * Gửi email với file đính kèm
   */
  async sendEmailWithAttachment(
    to: string,
    subject: string,
    html: string,
    attachments: Array<{ filename: string; path: string }>,
  ): Promise<void> {
    await this.mailerService.sendMail({
      to,
      subject,
      html,
      attachments,
    });
  }

  /**
   * Gửi email hàng loạt (batch)
   */
  async sendBulkEmails(recipients: Array<{ email: string; name?: string }>, subject: string, template: string, context: any): Promise<void> {
    const promises = recipients.map(recipient =>
      this.sendTemplateEmail({
        to: recipient.email,
        subject,
        template,
        context: {
          ...context,
          name: recipient.name || 'User',
        },
      }),
    );

    await Promise.allSettled(promises);
    this.logger.log(`Bulk emails sent to ${recipients.length} recipients`);
  }
}
```

### 5. **Mail Templates (Handlebars)**

```handlebars
<!-- templates/otp.hbs -->
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <style>
    body {
      font-family: Arial, sans-serif;
      line-height: 1.6;
      color: #333;
    }
    .container {
      max-width: 600px;
      margin: 0 auto;
      padding: 20px;
      border: 1px solid #ddd;
      border-radius: 5px;
    }
    .otp-code {
      font-size: 32px;
      font-weight: bold;
      color: #4F46E5;
      text-align: center;
      padding: 20px;
      background: #F3F4F6;
      border-radius: 5px;
      letter-spacing: 5px;
    }
    .warning {
      color: #EF4444;
      font-size: 14px;
      margin-top: 20px;
    }
    .footer {
      margin-top: 30px;
      font-size: 12px;
      color: #6B7280;
      text-align: center;
    }
  </style>
</head>
<body>
  <div class="container">
    <h2>Xác thực tài khoản</h2>
    <p>Xin chào <strong>{{name}}</strong>,</p>
    <p>Cảm ơn bạn đã đăng ký tài khoản. Vui lòng sử dụng mã OTP dưới đây để hoàn tất đăng ký:</p>
    
    <div class="otp-code">
      {{otp}}
    </div>
    
    <p>Mã OTP này có hiệu lực trong vòng <strong>{{expiryMinutes}} phút</strong>.</p>
    
    <div class="warning">
      ⚠️ Nếu bạn không yêu cầu mã này, vui lòng bỏ qua email này.
    </div>
    
    <div class="footer">
      <p>&copy; {{year}} Your Company. All rights reserved.</p>
    </div>
  </div>
</body>
</html>
```

```handlebars
<!-- templates/welcome.hbs -->
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <style>
    .container {
      max-width: 600px;
      margin: 0 auto;
      padding: 20px;
    }
    .button {
      display: inline-block;
      padding: 10px 20px;
      background-color: #4F46E5;
      color: white;
      text-decoration: none;
      border-radius: 5px;
      margin-top: 20px;
    }
  </style>
</head>
<body>
  <div class="container">
    <h1>Chào mừng {{name}}! 🎉</h1>
    <p>Cảm ơn bạn đã đăng ký tài khoản thành công.</p>
    <p>Bạn có thể đăng nhập ngay bây giờ để trải nghiệm các tính năng:</p>
    <a href="{{loginUrl}}" class="button">Đăng nhập ngay</a>
    <hr />
    <p>Mọi thắc mắc xin liên hệ support@yourapp.com</p>
  </div>
</body>
</html>
```

```handlebars
<!-- templates/reset-password.hbs -->
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <style>
    .container {
      max-width: 600px;
      margin: 0 auto;
      padding: 20px;
    }
    .button {
      display: inline-block;
      padding: 12px 24px;
      background-color: #EF4444;
      color: white;
      text-decoration: none;
      border-radius: 5px;
      margin: 20px 0;
    }
    .expiry {
      color: #6B7280;
      font-size: 14px;
    }
  </style>
</head>
<body>
  <div class="container">
    <h2>Đặt lại mật khẩu</h2>
    <p>Xin chào <strong>{{name}}</strong>,</p>
    <p>Chúng tôi nhận được yêu cầu đặt lại mật khẩu cho tài khoản của bạn.</p>
    
    <a href="{{resetUrl}}" class="button">Đặt lại mật khẩu</a>
    
    <p class="expiry">Link này có hiệu lực trong vòng {{expiryHours}} giờ.</p>
    
    <p>Nếu bạn không yêu cầu đặt lại mật khẩu, vui lòng bỏ qua email này.</p>
    
    <hr />
    <p style="font-size: 12px; color: #6B7280;">
      &copy; {{year}} Your Company. All rights reserved.
    </p>
  </div>
</body>
</html>
```

### 6. **Mail Controller (Tuỳ chọn - để test)**

```typescript
// modules/mail/mail.controller.ts
import { Controller, Post, Body, UseGuards } from '@nestjs/common';
import { MailService } from './mail.service';

@Controller('mail')
export class MailController {
  constructor(private readonly mailService: MailService) {}

  @Post('send-otp')
  async sendOtp(@Body() body: { email: string; otp: string; name?: string }) {
    await this.mailService.sendOtpEmail(body.email, body.otp, body.name);
    return { success: true, message: 'OTP email sent successfully' };
  }

  @Post('send-welcome')
  async sendWelcome(@Body() body: { email: string; name: string }) {
    await this.mailService.sendWelcomeEmail(body.email, body.name);
    return { success: true, message: 'Welcome email sent successfully' };
  }

  @Post('send-reset-password')
  async sendResetPassword(@Body() body: { email: string; resetToken: string; name?: string }) {
    await this.mailService.sendResetPasswordEmail(body.email, body.resetToken, body.name);
    return { success: true, message: 'Reset password email sent successfully' };
  }
}
```

### 7. **Tích hợp vào App Module**

```typescript
// app.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { MailModule } from './modules/mail/mail.module';
import { AuthModule } from './modules/auth/auth.module';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      envFilePath: '.env',
    }),
    MailModule,
    AuthModule,
  ],
})
export class AppModule {}
```

### 8. **Sử dụng trong Auth Service**

```typescript
// modules/auth/auth.service.ts
import { Injectable } from '@nestjs/common';
import { MailService } from '../mail/mail.service';

@Injectable()
export class AuthService {
  constructor(
    private readonly mailService: MailService,
  ) {}

  async sendOtpRegister(email: string, username: string) {
    // Generate OTP
    const otp = Math.floor(100000 + Math.random() * 900000).toString();
    
    // Save OTP to database/cache
    await this.saveOtpToCache(email, otp);
    
    // Send OTP email
    await this.mailService.sendOtpEmail(email, otp, username);
    
    return {
      success: true,
      message: 'OTP đã được gửi đến email của bạn',
    };
  }

  async sendWelcomeAfterRegister(user: any) {
    await this.mailService.sendWelcomeEmail(user.email, user.username);
  }
}
```

## 🔐 Cấu hình cho các Email Provider phổ biến

### **Gmail**
```env
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USER=your-email@gmail.com
MAIL_PASSWORD=your-app-specific-password
MAIL_SECURE=false
```

### **SendGrid**
```env
MAIL_HOST=smtp.sendgrid.net
MAIL_PORT=587
MAIL_USER=apikey
MAIL_PASSWORD=your-sendgrid-api-key
MAIL_SECURE=false
```

### **Mailgun**
```env
MAIL_HOST=smtp.mailgun.org
MAIL_PORT=587
MAIL_USER=postmaster@your-domain.com
MAIL_PASSWORD=your-mailgun-password
MAIL_SECURE=false
```

### **Outlook/Hotmail**
```env
MAIL_HOST=smtp-mail.outlook.com
MAIL_PORT=587
MAIL_USER=your-email@outlook.com
MAIL_PASSWORD=your-password
MAIL_SECURE=false
```

## 🚀 Chạy ứng dụng

```bash
# Development
pnpm run start:dev

# Production
pnpm run build
pnpm run start:prod
```

## 📝 Test nhanh với curl

```bash
# Test send OTP
curl -X POST http://localhost:3000/mail/send-otp \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","otp":"123456","name":"John"}'

# Test send welcome
curl -X POST http://localhost:3000/mail/send-welcome \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","name":"John"}'
```

## ⚠️ Lưu ý quan trọng

1. **App Password cho Gmail**: Từ 2022, Gmail yêu cầu dùng "App Password" thay vì mật khẩu thật
2. **Rate Limiting**: Hầu hết SMTP providers có giới hạn số lượng email/ngày
3. **Queue**: Nên dùng queue (Bull, RabbitMQ) cho việc gửi email hàng loạt
4. **Retry Logic**: Thêm retry mechanism khi gửi thất bại
5. **Logging**: Luôn log để debug khi có lỗi
6. **Testing**: Dùng Ethereal Email cho môi trường test

Với hướng dẫn này, bạn đã có một hệ thống gửi email hoàn chỉnh trong NestJS sử dụng pnpm! 🎉