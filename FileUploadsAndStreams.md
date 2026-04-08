# Axios: File Uploads & Streams Guide

This guide covers handling binary data, uploading files, tracking progress, and working with streams in both browser and Node.js environments using Axios.

---

## 1. Uploading Files (FormData)

To send files from the browser, you must use the standard HTML `FormData` API. Axios automatically handles the necessary `Content-Type: multipart/form-data` headers and boundary generation when it detects a `FormData` payload.

```javascript
import axios from 'axios';

const uploadFile = async (file, metadata) => {
  const formData = new FormData();
  
  // Append the selected file (e.g., from an <input type="file" />)
  formData.append('document', file);
  
  // You can also append string metadata in the same request
  formData.append('description', metadata.description);
  formData.append('userId', metadata.userId);

  try {
    const response = await axios.post('/api/upload', formData, {
      // Note: You do NOT explicitly need to set the Content-Type header. 
      // Axios does this automatically for FormData.
      headers: {
        // Only set this explicitly if your global interceptors override it 
        // with something else like application/json
        'Content-Type': 'multipart/form-data'
      }
    });
    console.log('Upload successful!', response.data);
  } catch (error) {
    console.error('Upload failed', error);
  }
}
```

---

## 2. Tracking Upload Progress

A crucial feature for file uploads is providing feedback to the user. Axios provides the `onUploadProgress` callback in the request configuration.

```javascript
const uploadWithProgress = async (file) => {
  const formData = new FormData();
  formData.append('file', file);

  try {
    await axios.post('/api/upload', formData, {
      onUploadProgress: (progressEvent) => {
        // Calculate the percentage
        const percentCompleted = Math.round(
          (progressEvent.loaded * 100) / progressEvent.total
        );
        
        console.log(`Upload progress: ${percentCompleted}%`);
        
        // Example: Update React state here
        // setProgress(percentCompleted);
        
        // Additionally, Axios 1.4.0+ provides an `estimated` and `rate` 
        // property in the progress event
        if (progressEvent.estimated) {
           console.log(`Estimated time remaining: ${progressEvent.estimated}s`);
        }
      }
    });
  } catch (error) {
    // Handle error
  }
}
```

---

## 3. Canceling Uploads/Downloads

Large file transfers might need to be voluntarily canceled by the user (e.g., if it's taking too long on a bad network). You can achieve this using the native `AbortController` API, which Axios fully supports.

```javascript
import { useRef } from 'react';

const UploadComponent = () => {
  const abortControllerRef = useRef(null);

  const startUpload = async (file) => {
    // Create a new controller for this request
    abortControllerRef.current = new AbortController();
    
    const formData = new FormData();
    formData.append('data', file);

    try {
      await axios.post('/api/upload', formData, {
        signal: abortControllerRef.current.signal, // Pass the signal here
      });
      console.log('Upload completed.');
    } catch (error) {
      if (axios.isCancel(error)) {
        console.log('Request canceled intentionally by the user.');
      } else {
        console.error('Upload failed due to network error.', error);
      }
    }
  };

  const cancelUpload = () => {
    if (abortControllerRef.current) {
      // Triggers cancellation
      abortControllerRef.current.abort();
    }
  };

  return (
    <div>
       <button onClick={() => startUpload(myFile)}>Upload</button>
       <button onClick={cancelUpload}>Cancel</button>
    </div>
  )
}
```

---

## 4. Downloading Files (Browser)

When downloading binary files in the browser via an API endpoint, you must tell Axios to expect binary data using `responseType: 'blob'`. Without this, Axios will try to parse the file as a JSON string, corrupting the downloaded file.

```javascript
const downloadReport = async (reportId) => {
  try {
    const response = await axios.get(`/api/reports/${reportId}/download`, {
      // Critical! Treats the response as binary data instead of string/JSON
      responseType: 'blob', 
      
      // Track download progress
      onDownloadProgress: (progressEvent) => {
         const percentage = Math.round((progressEvent.loaded * 100) / progressEvent.total);
         console.log(`Downloading: ${percentage}%`);
      }
    });

    // 1. Create a dynamic URL for the binary Blob
    const url = window.URL.createObjectURL(new Blob([response.data]));
    
    // 2. Create a temporary anchor element
    const link = document.createElement('a');
    link.href = url;
    
    // 3. Attempt to extract the correct file name from headers
    const contentDisposition = response.headers['content-disposition'];
    let fileName = 'downloaded_file.pdf'; // Fallback
    
    if (contentDisposition) {
      const match = contentDisposition.match(/filename="(.+)"/);
      if (match && match.length > 1) fileName = match[1];
    }
    
    link.setAttribute('download', fileName);
    
    // 4. Trigger download and cleanup
    document.body.appendChild(link);
    link.click();
    document.body.removeChild(link);
    window.URL.revokeObjectURL(url);
    
  } catch (error) {
    if (error.response?.data instanceof Blob) {
      // Edge case: Sometimes the backend returns a JSON error inside a Blob payload.
      // You must manually parse the Blob to read the error message.
      const text = await error.response.data.text();
      try {
        const errorData = JSON.parse(text);
        console.error("Backend Error:", errorData.message);
      } catch (parseErr) {
        console.error("Failed to parse inner Blob error.");
      }
    }
  }
}
```

---

## 5. Streams in Node.js Environments

If you are using Axios on the server (e.g., Next.js API Routes, Express, NestJS), you can stream large files to handle massive datasets without consuming memory.

By setting `responseType: 'stream'`, Axios returns a Node.js Readable Stream rather than loading the whole payload into memory buffers.

```javascript
/* === NODE.JS ENVIRONMENT ONLY === */
const fs = require('fs');
const path = require('path');
const axios = require('axios');

async function downloadGenericFileToDisk(url, destinationFilename) {
  const filePath = path.resolve(__dirname, 'downloads', destinationFilename);
  const writer = fs.createWriteStream(filePath);

  const response = await axios({
    url,
    method: 'GET',
    responseType: 'stream' // We want a stream, not holding the whole file in memory
  });

  // Pipe the incoming network stream straight to disk
  response.data.pipe(writer);

  return new Promise((resolve, reject) => {
    // Stream successfully finished writing
    writer.on('finish', resolve);
    
    // An error occurred during writing
    writer.on('error', (err) => {
      fs.unlink(filePath, () => {}); // cleanup partial file
      reject(err);
    });
  });
}
```

### Uploading streams (Node.js)

Similarly, you can stream files from the local filesystem to an external API without loading the entire file into Node's memory. This is particularly useful when proxying gigabyte-sized files.

```javascript
/* === NODE.JS ENVIRONMENT ONLY === */
const fs = require('fs');
const axios = require('axios');
const FormData = require('form-data'); // Needs the `form-data` package in Node

async function uploadStreamFile() {
  const form = new FormData();
  
  // Passing a stream directly into form-data
  form.append('my_field', fs.createReadStream('/foo/bar.jpg'));

  await axios.post('https://example.com/upload', form, {
    // Must explicitly set headers from the form-data package in Node
    headers: {
      ...form.getHeaders()
    }
  });
}
```

--- 

## Summary Checklist
- [x] Use `FormData` with native JS `File` or `Blob` objects for uploading.
- [x] Avoid forcing `Content-Type: multipart/form-data` explicitly; let Axios handle the boundary configurations.
- [x] Use `onUploadProgress` / `onDownloadProgress` to update user interfaces.
- [x] For browser file downloads, **ALWAYS** set `responseType: 'blob'`.
- [x] Pass an `AbortController` signal to safely cancel large active transfers.
- [x] In Node.js server environments, use `responseType: 'stream'` to pipe data effectively and save memory.
