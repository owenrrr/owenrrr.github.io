---
layout: post
title: Image Hosting Website
feature-img: "assets/img/portfolio/imagehost.png"
img: "assets/img/portfolio/imagehost.png"
date: 15 Feb 2026
tags: [Project]
---
# Intro
This is a website that provides the functionality of image hosting. Currently it's free and for educational use. It's very simple to use. You could upload an image(jpg,png.etc) and the website would provide an URL link to you. It's viewable for everyone who got this URL so keeps it secret if you do not want others to access the image. You could visit the website through [Image Hosting Website](https://image.host.owenrrr.me).

# Usage
![What it looks like](/assets/img/portfolio/imagehost.png)

# Image Hosting Website
## Structure
```bash
┌──────────────────────────────────────────────────────────────────────┐
│                              User Browser                            │
│  React(Vite) App.tsx                                                 │
│  - Choose & Preview file (blob URL)                                  │
│  - Call Backend: /uploads/init, /uploads/complete                    │
│  - PUT S3 bucket (presigned PUT URL)                                 │
│  - Show public view URL (permanent)                                  │
└───────────────┬───────────────────────────────────────┬──────────────┘
                │ HTTPS (API_BASE)                      │ HTTPS (PUT)
                │                                       │ (preflight OPTIONS)
                v                                       v
     ┌───────────────────────┐               ┌───────────────────────────┐
     │     Backend API       │               │      S3 Image Bucket      │
     │  C++ Drogon + AWS SDK │               │  - uploads/<uuid> object  │
     │  /uploads/init        │               │  - CORS must allow PUT    │
     │   -> presigned PUT URL│               │  - Bucket policy: public  │
     │  /uploads/complete    │               │    read uploads/*         │
     │   -> public view URL  │               └───────────┬───────────────┘
     │  /version /health     │                           │ HTTPS (GET)
     └───────────┬───────────┘                           │ (no signature)
                 │ IAM Role (ECS task role)              v
                 │ AWS creds to sign presigned URL  ┌──────────────────────┐
                 v                                  │ Anyone with the URL  │
        ┌──────────────────┐                        │ can view image       │
        │ AWS SDK S3Client │                        └──────────────────────┘
        │ GeneratePresigned│
        └──────────────────┘
```

## Techs & Tools
- Frontend: React + Vite
- Backend: Drogon
- Frontend Website Hosting: Cloudfront
- Backend Deployment: ECR + ECS

## Process

1. User uploads the image
2. Frontend sends a request to the backend (`/uploads/init`)
3. Backend responds to the frontend with presigned URL
4. Frontend sends OPTIONS preflight to the S3 bucket (CORS)
5. S3 bucket returns CORS OK
6. Frontend sends a PUT (uploadURL + file) to the S3 bucket
7. S3 bucket returns success
8. Frontend sends a request (with URL) to the backend to represent this upload is complete (`/uploads/complete`)
9. Backend returns `viewURL`
