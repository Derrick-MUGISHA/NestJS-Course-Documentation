# Module 8: File Handling & Media

## 📚 Table of Contents
1. [Overview](#overview)
2. [File Upload Setup](#file-upload-setup)
3. [Basic File Upload](#basic-file-upload)
4. [File Validation](#file-validation)
5. [Multiple File Upload](#multiple-file-upload)
6. [File Storage Options](#file-storage-options)
7. [AWS S3 Integration](#aws-s3-integration)
8. [Image Processing](#image-processing)
9. [File Download](#file-download)
10. [Streaming Files](#streaming-files)
11. [Best Practices](#best-practices)
12. [Mini Project: Media Management System](#mini-project-media-management-system)

---

## Overview

This module covers handling file uploads, downloads, and media processing in NestJS applications. You'll learn to upload files, validate them, store them securely, and integrate with cloud storage services.

**Topics Covered:**
- File upload handling
- File validation and security
- Local and cloud storage
- Image processing
- File streaming
- Download handling

---

## File Upload Setup

### Install Dependencies

```bash
npm install @nestjs/platform-express multer
npm install -D @types/multer
```

**Why Multer?**
- Multer is a middleware for handling `multipart/form-data` (file uploads)
- It's the standard for file uploads in Express/NestJS
- Handles file streams efficiently
- Supports multiple files and custom storage

### Configure Multer

Multer is configured per-route, but we'll set up reusable configurations:

```typescript
// src/config/multer.config.ts
import { MulterOptions } from '@nestjs/platform-express/multer/interfaces/multer-options.interface';
import { diskStorage } from 'multer';
import { extname } from 'path';
import { existsSync, mkdirSync } from 'fs';

// Helper function to create upload directories if they don't exist
function createUploadDir(path: string) {
  if (!existsSync(path)) {
    mkdirSync(path, { recursive: true });
  }
}

// Configuration for image uploads
export const imageUploadConfig: MulterOptions = {
  // diskStorage stores files on the local filesystem
  // For production, consider using memoryStorage() and uploading to S3
  storage: diskStorage({
    // destination: Where to store uploaded files
    destination: (req, file, cb) => {
      const uploadPath = './uploads/images';
      createUploadDir(uploadPath);
      cb(null, uploadPath);
    },
    
    // filename: How to name the uploaded file
    filename: (req, file, cb) => {
      // Generate unique filename to prevent collisions
      const uniqueSuffix = Date.now() + '-' + Math.round(Math.random() * 1e9);
      
      // Get original file extension
      const ext = extname(file.originalname);
      
      // Create filename: timestamp-random.extension
      const filename = `${file.fieldname}-${uniqueSuffix}${ext}`;
      cb(null, filename);
    },
  }),
  
  // fileFilter: Validate files before accepting them
  fileFilter: (req, file, cb) => {
    // Only allow image files
    if (file.mimetype.match(/\/(jpg|jpeg|png|gif|webp)$/)) {
      cb(null, true); // Accept file
    } else {
      cb(new Error('Only image files are allowed!'), false); // Reject file
    }
  },
  
  // limits: Set file size and count limits
  limits: {
    fileSize: 5 * 1024 * 1024, // 5MB max file size
    files: 1, // Maximum 1 file per request
  },
};

// Configuration for document uploads
export const documentUploadConfig: MulterOptions = {
  storage: diskStorage({
    destination: './uploads/documents',
    filename: (req, file, cb) => {
      const uniqueSuffix = Date.now() + '-' + Math.round(Math.random() * 1e9);
      const ext = extname(file.originalname);
      const filename = `${file.fieldname}-${uniqueSuffix}${ext}`;
      cb(null, filename);
    },
  }),
  fileFilter: (req, file, cb) => {
    // Allow PDF, Word, Excel files
    const allowedMimes = [
      'application/pdf',
      'application/msword',
      'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
      'application/vnd.ms-excel',
      'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
    ];
    
    if (allowedMimes.includes(file.mimetype)) {
      cb(null, true);
    } else {
      cb(new Error('Invalid file type. Only PDF, Word, and Excel files are allowed.'), false);
    }
  },
  limits: {
    fileSize: 10 * 1024 * 1024, // 10MB for documents
  },
};
```

---

## Basic File Upload

### Single File Upload

Handle a single file upload with validation:

```typescript
// src/files/files.controller.ts
import {
  Controller,
  Post,
  UseInterceptors,
  UploadedFile,
  ParseFilePipe,
  MaxFileSizeValidator,
  FileTypeValidator,
} from '@nestjs/common';
import { FileInterceptor } from '@nestjs/platform-express';
import { FilesService } from './files.service';

@Controller('files')
export class FilesController {
  constructor(private readonly filesService: FilesService) {}

  // @Post('upload') - Endpoint for file upload
  // @UseInterceptors(FileInterceptor('file')) - Interceptor for single file
  // 'file' is the field name in the form data
  @Post('upload')
  @UseInterceptors(FileInterceptor('file'))
  async uploadFile(
    // @UploadedFile() extracts the uploaded file from the request
    // ParseFilePipe validates the file before it reaches the handler
    @UploadedFile(
      new ParseFilePipe({
        // validators: Array of validators to check
        validators: [
          // MaxFileSizeValidator: Check file size
          // 5MB maximum
          new MaxFileSizeValidator({ maxSize: 5 * 1024 * 1024 }),
          
          // FileTypeValidator: Check file type by MIME type
          // Only allow image MIME types
          new FileTypeValidator({ fileType: /(jpg|jpeg|png|gif|webp)/ }),
        ],
      }),
    )
    file: Express.Multer.File,
  ) {
    // file object contains:
    // - fieldname: The form field name
    // - originalname: Original filename
    // - encoding: File encoding
    // - mimetype: MIME type (e.g., 'image/jpeg')
    // - size: File size in bytes
    // - destination: Where file was saved
    // - filename: Generated filename
    // - path: Full path to the file
    
    console.log('File uploaded:', {
      originalName: file.originalname,
      filename: file.filename,
      size: file.size,
      mimetype: file.mimetype,
      path: file.path,
    });

    // Save file metadata to database
    const fileRecord = await this.filesService.create({
      originalName: file.originalname,
      filename: file.filename,
      path: file.path,
      size: file.size,
      mimetype: file.mimetype,
    });

    return {
      message: 'File uploaded successfully',
      file: fileRecord,
      url: `/files/${fileRecord.id}`, // URL to access the file
    };
  }
}
```

**Understanding the File Object:**
- `originalname`: The name provided by the user
- `filename`: The generated unique filename
- `path`: Full filesystem path
- `mimetype`: Detected file type
- `size`: File size in bytes

### File Service

```typescript
// src/files/files.service.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { File } from './entities/file.entity';

@Injectable()
export class FilesService {
  constructor(
    @InjectRepository(File)
    private filesRepository: Repository<File>,
  ) {}

  // Create file record in database
  async create(fileData: {
    originalName: string;
    filename: string;
    path: string;
    size: number;
    mimetype: string;
  }): Promise<File> {
    const file = this.filesRepository.create(fileData);
    return await this.filesRepository.save(file);
  }

  // Find file by ID
  async findOne(id: number): Promise<File> {
    return await this.filesRepository.findOne({ where: { id } });
  }

  // Delete file from filesystem and database
  async remove(id: number): Promise<void> {
    const file = await this.findOne(id);
    
    if (file) {
      // Delete from filesystem
      const fs = require('fs');
      if (fs.existsSync(file.path)) {
        fs.unlinkSync(file.path);
      }
      
      // Delete from database
      await this.filesRepository.delete(id);
    }
  }
}
```

---

## File Validation

### Custom Validators

Create custom validators for complex validation:

```typescript
// src/files/validators/file-validator.pipe.ts
import {
  PipeTransform,
  Injectable,
  ArgumentMetadata,
  BadRequestException,
} from '@nestjs/common';

@Injectable()
export class FileValidatorPipe implements PipeTransform {
  transform(value: Express.Multer.File, metadata: ArgumentMetadata) {
    // Check file exists
    if (!value) {
      throw new BadRequestException('No file uploaded');
    }

    // Validate file size (5MB max)
    const maxSize = 5 * 1024 * 1024;
    if (value.size > maxSize) {
      throw new BadRequestException(`File size exceeds ${maxSize / 1024 / 1024}MB`);
    }

    // Validate file type
    const allowedMimeTypes = ['image/jpeg', 'image/png', 'image/gif', 'image/webp'];
    if (!allowedMimeTypes.includes(value.mimetype)) {
      throw new BadRequestException('Invalid file type. Only images are allowed.');
    }

    // Validate filename (prevent path traversal attacks)
    if (value.originalname.includes('..') || value.originalname.includes('/')) {
      throw new BadRequestException('Invalid filename');
    }

    return value;
  }
}

// Usage in controller
@Post('upload')
@UseInterceptors(FileInterceptor('file'))
async uploadFile(
  @UploadedFile(FileValidatorPipe) file: Express.Multer.File,
) {
  // File is validated at this point
  return this.filesService.create(file);
}
```

### File Type Detection

Detect file type by content, not just extension:

```typescript
// src/files/file-type-detector.service.ts
import { Injectable } from '@nestjs/common';
import * as fileType from 'file-type';
import * as fs from 'fs';

@Injectable()
export class FileTypeDetectorService {
  async detectFileType(filePath: string): Promise<string | null> {
    // Read first bytes of file
    const buffer = fs.readFileSync(filePath).slice(0, 4100);
    
    // Detect MIME type from file content (more reliable than extension)
    const type = await fileType.fromBuffer(buffer);
    
    return type?.mime || null;
  }

  async validateImageType(filePath: string): Promise<boolean> {
    const mimeType = await this.detectFileType(filePath);
    const allowedTypes = ['image/jpeg', 'image/png', 'image/gif', 'image/webp'];
    return allowedTypes.includes(mimeType);
  }
}
```

---

## Multiple File Upload

Handle multiple files in a single request:

```typescript
// src/files/files.controller.ts
import { FilesInterceptor } from '@nestjs/platform-express';

@Controller('files')
export class FilesController {
  // Upload multiple files
  @Post('upload-multiple')
  // FilesInterceptor('files', { maxCount: 10 }) - Allow up to 10 files
  // 'files' is the field name in form data
  @UseInterceptors(FilesInterceptor('files', { maxCount: 10 }))
  async uploadMultipleFiles(
    // @UploadedFiles() (plural) extracts array of files
    @UploadedFiles() files: Array<Express.Multer.File>,
  ) {
    const uploadedFiles = [];

    // Process each file
    for (const file of files) {
      const fileRecord = await this.filesService.create({
        originalName: file.originalname,
        filename: file.filename,
        path: file.path,
        size: file.size,
        mimetype: file.mimetype,
      });

      uploadedFiles.push(fileRecord);
    }

    return {
      message: `${files.length} files uploaded successfully`,
      files: uploadedFiles,
    };
  }

  // Upload multiple files with different field names
  @Post('upload-mixed')
  // Use FileFieldsInterceptor for multiple fields
  @UseInterceptors(
    FileFieldsInterceptor([
      { name: 'avatar', maxCount: 1 },
      { name: 'documents', maxCount: 5 },
    ]),
  )
  async uploadMixedFiles(
    @UploadedFiles()
    files: {
      avatar?: Express.Multer.File[];
      documents?: Express.Multer.File[];
    },
  ) {
    const results = {};

    if (files.avatar) {
      results['avatar'] = await this.filesService.create(files.avatar[0]);
    }

    if (files.documents) {
      results['documents'] = await Promise.all(
        files.documents.map(file => this.filesService.create(file)),
      );
    }

    return results;
  }
}
```

---

## File Storage Options

### Local Storage

Store files on the server's filesystem:

```typescript
// Pros: Simple, no external dependencies
// Cons: Not scalable, limited by server storage
const localStorage = diskStorage({
  destination: './uploads',
  filename: (req, file, cb) => {
    const uniqueName = `${Date.now()}-${file.originalname}`;
    cb(null, uniqueName);
  },
});
```

### Memory Storage

Store files in memory (useful for processing before uploading to cloud):

```typescript
// Files are stored in memory as Buffer
// Useful when you need to process before storing
import { memoryStorage } from 'multer';

const memoryStorageConfig = memoryStorage();
```

### Cloud Storage

Store files in cloud services (S3, Cloudinary, etc.) - See AWS S3 section below.

---

## AWS S3 Integration

AWS S3 provides scalable, secure cloud storage for files.

### Install AWS SDK

```bash
npm install @aws-sdk/client-s3 @aws-sdk/s3-request-presigner
```

### Configure S3 Service

```typescript
// src/files/s3.service.ts
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { S3Client, PutObjectCommand, GetObjectCommand, DeleteObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import * as fs from 'fs';

@Injectable()
export class S3Service {
  private s3Client: S3Client;
  private bucketName: string;

  constructor(private configService: ConfigService) {
    // Initialize S3 client with credentials
    this.s3Client = new S3Client({
      region: this.configService.get('AWS_REGION'),
      credentials: {
        accessKeyId: this.configService.get('AWS_ACCESS_KEY_ID'),
        secretAccessKey: this.configService.get('AWS_SECRET_ACCESS_KEY'),
      },
    });

    this.bucketName = this.configService.get('AWS_S3_BUCKET');
  }

  // Upload file to S3
  async uploadFile(
    filePath: string,
    key: string,
    contentType: string,
  ): Promise<string> {
    // Read file from local filesystem
    const fileContent = fs.readFileSync(filePath);

    // Create PutObject command
    const command = new PutObjectCommand({
      Bucket: this.bucketName,
      Key: key, // S3 object key (path/filename)
      Body: fileContent,
      ContentType: contentType,
      // ACL: 'public-read', // Make file publicly accessible (if needed)
    });

    // Upload to S3
    await this.s3Client.send(command);

    // Return S3 URL
    return `https://${this.bucketName}.s3.amazonaws.com/${key}`;
  }

  // Generate pre-signed URL for private file access
  // Pre-signed URLs allow temporary access without making bucket public
  async getSignedUrl(key: string, expiresIn: number = 3600): Promise<string> {
    const command = new GetObjectCommand({
      Bucket: this.bucketName,
      Key: key,
    });

    // Generate URL that expires in specified seconds
    const url = await getSignedUrl(this.s3Client, command, { expiresIn });
    return url;
  }

  // Delete file from S3
  async deleteFile(key: string): Promise<void> {
    const command = new DeleteObjectCommand({
      Bucket: this.bucketName,
      Key: key,
    });

    await this.s3Client.send(command);
  }
}
```

### Upload to S3

```typescript
// src/files/files.controller.ts
@Post('upload-to-s3')
@UseInterceptors(FileInterceptor('file'))
async uploadToS3(
  @UploadedFile() file: Express.Multer.File,
) {
  // Generate unique key for S3
  const key = `uploads/${Date.now()}-${file.originalname}`;

  // Upload to S3
  const s3Url = await this.s3Service.uploadFile(
    file.path,
    key,
    file.mimetype,
  );

  // Delete local file after upload (optional)
  fs.unlinkSync(file.path);

  // Save metadata to database
  const fileRecord = await this.filesService.create({
    originalName: file.originalname,
    filename: key, // Store S3 key, not local filename
    path: s3Url, // Store S3 URL
    size: file.size,
    mimetype: file.mimetype,
    storage: 's3', // Indicate storage location
  });

  return {
    message: 'File uploaded to S3 successfully',
    file: fileRecord,
    url: s3Url,
  };
}
```

---

## Image Processing

Process and optimize images after upload.

### Install Image Processing Library

```bash
npm install sharp
```

### Image Processing Service

```typescript
// src/files/image-processing.service.ts
import { Injectable } from '@nestjs/common';
import * as sharp from 'sharp';

@Injectable()
export class ImageProcessingService {
  // Resize image to specific dimensions
  async resizeImage(
    inputPath: string,
    outputPath: string,
    width: number,
    height: number,
  ): Promise<void> {
    await sharp(inputPath)
      .resize(width, height, {
        fit: 'inside', // Maintain aspect ratio
        withoutEnlargement: true, // Don't enlarge small images
      })
      .jpeg({ quality: 85 }) // Compress to JPEG with 85% quality
      .toFile(outputPath);
  }

  // Generate thumbnail
  async generateThumbnail(
    inputPath: string,
    outputPath: string,
    size: number = 300,
  ): Promise<void> {
    await sharp(inputPath)
      .resize(size, size, {
        fit: 'cover', // Crop to fill dimensions
        position: 'center',
      })
      .jpeg({ quality: 80 })
      .toFile(outputPath);
  }

  // Compress image
  async compressImage(
    inputPath: string,
    outputPath: string,
    quality: number = 85,
  ): Promise<void> {
    await sharp(inputPath)
      .jpeg({ quality })
      .toFile(outputPath);
  }

  // Process image: resize, compress, and generate thumbnail
  async processImage(inputPath: string, baseOutputPath: string): Promise<{
    original: string;
    resized: string;
    thumbnail: string;
  }> {
    const resizedPath = `${baseOutputPath}-resized.jpg`;
    const thumbnailPath = `${baseOutputPath}-thumb.jpg`;

    // Process in parallel for better performance
    await Promise.all([
      this.resizeImage(inputPath, resizedPath, 1920, 1080),
      this.generateThumbnail(inputPath, thumbnailPath, 300),
    ]);

    return {
      original: inputPath,
      resized: resizedPath,
      thumbnail: thumbnailPath,
    };
  }
}
```

### Process Images on Upload

```typescript
@Post('upload-image')
@UseInterceptors(FileInterceptor('image'))
async uploadAndProcessImage(
  @UploadedFile() file: Express.Multer.File,
) {
  // Process image: create resized version and thumbnail
  const processed = await this.imageProcessingService.processImage(
    file.path,
    file.path.replace('.jpg', ''), // Base path without extension
  );

  // Save all versions to database
  const fileRecord = await this.filesService.create({
    originalName: file.originalname,
    filename: file.filename,
    path: processed.resized, // Store resized version
    thumbnailPath: processed.thumbnail,
    size: file.size,
    mimetype: file.mimetype,
  });

  return {
    message: 'Image uploaded and processed',
    file: fileRecord,
    urls: {
      original: `/files/${fileRecord.id}`,
      thumbnail: `/files/${fileRecord.id}/thumbnail`,
  },
};
```

---

## File Download

Serve files securely with proper headers:

```typescript
// src/files/files.controller.ts
import { Res, StreamableFile } from '@nestjs/common';
import { Response } from 'express';
import { createReadStream } from 'fs';
import { join } from 'path';

@Get(':id')
async getFile(
  @Param('id') id: string,
  @Res({ passthrough: true }) res: Response,
) {
  const file = await this.filesService.findOne(+id);

  if (!file) {
    throw new NotFoundException('File not found');
  }

  // Create readable stream
  const fileStream = createReadStream(join(process.cwd(), file.path));

  // Set response headers
  res.set({
    'Content-Type': file.mimetype,
    'Content-Disposition': `attachment; filename="${file.originalName}"`,
    'Content-Length': file.size,
  });

  // Return streamable file
  return new StreamableFile(fileStream);
}

// Download with custom filename
@Get(':id/download')
async downloadFile(
  @Param('id') id: string,
  @Res() res: Response,
) {
  const file = await this.filesService.findOne(+id);

  if (!file) {
    throw new NotFoundException('File not found');
  }

  res.download(file.path, file.originalName, (err) => {
    if (err) {
      throw new InternalServerErrorException('Error downloading file');
    }
  });
}
```

---

## Streaming Files

Stream large files efficiently:

```typescript
@Get(':id/stream')
async streamFile(
  @Param('id') id: string,
  @Res() res: Response,
) {
  const file = await this.filesService.findOne(+id);

  if (!file) {
    throw new NotFoundException('File not found');
  }

  // Create read stream
  const stream = createReadStream(file.path);

  // Set headers for streaming
  res.setHeader('Content-Type', file.mimetype);
  res.setHeader('Content-Length', file.size);

  // Pipe stream to response
  stream.pipe(res);
}
```

---

## Best Practices

### 1. Validate File Types

Always validate by MIME type, not just extension (extensions can be spoofed).

### 2. Set File Size Limits

```typescript
limits: {
  fileSize: 5 * 1024 * 1024, // 5MB
}
```

### 3. Sanitize Filenames

```typescript
filename: (req, file, cb) => {
  // Remove special characters and sanitize
  const sanitizedName = file.originalname.replace(/[^a-zA-Z0-9.-]/g, '');
  cb(null, `${Date.now()}-${sanitizedName}`);
}
```

### 4. Use Cloud Storage in Production

Local storage doesn't scale - use S3 or similar for production.

### 5. Process Images Asynchronously

Use queues for image processing to avoid blocking requests.

---

## Mini Project: Media Management System

Build a complete media management system with:
1. Image upload with processing
2. File metadata storage
3. Thumbnail generation
4. S3 integration
5. File download with authentication
6. File deletion

---

## Next Steps

✅ Handle file uploads securely  
✅ Validate and process files  
✅ Integrate cloud storage  
✅ Process images efficiently  
✅ Move to Module 9: Caching with Redis  

---

*You now know how to handle files and media! 📁*

