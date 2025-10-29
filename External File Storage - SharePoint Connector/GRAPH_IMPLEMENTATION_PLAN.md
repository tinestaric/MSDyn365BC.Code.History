# SharePoint Graph API Implementation Plan
**Date:** October 29, 2025  
**Purpose:** Analyze feasibility and plan implementation of Graph API-based SharePoint connector

## ⚠️ CHUNKED UPLOAD/DOWNLOAD VERDICT: UPLOAD READY, DOWNLOAD LIMITED

**Upload:** ✅ Full chunked upload support via `UploadLargeFile()` - handles files up to **250GB** with **4MB chunks**  
**Download:** ⚠️ **CRITICAL LIMITATION** - BC HttpClient response max **150MB** - current implementation lacks chunked download  
**Status:** 🟡 **NEEDS ENHANCEMENT** - Chunked download must be implemented for files > 150MB!

### 🚨 Business Central Platform Limitation
Business Central's HttpClient has a hard limit of **150MB** for response content. This means:
- ❌ Files > 150MB **cannot be downloaded** with current implementation
- ⚠️ This affects both REST and Graph connectors
- ✅ Graph API **supports** Range requests for chunked download
- 🔧 **Solution Required:** Implement chunked download using HTTP Range headers

---

## Executive Summary

The Graph API module provides **sufficient functionality** to implement an alternative SharePoint connector. However, some **gaps and design considerations** need to be addressed.

---

## Current State Analysis

### Existing REST API Connector (`ExtSharePointConnectorImpl`)
The current implementation uses the SharePoint REST API with the following operations:

1. **File Operations**
   - `ListFiles()` - Get files in a folder
   - `GetFile()` - Download file by path
   - `CreateFile()` - Upload file
   - `CopyFile()` - Copy file (implemented via download + upload)
   - `MoveFile()` - Move file (implemented via download + upload + delete)
   - `FileExists()` - Check file existence
   - `DeleteFile()` - Delete file

2. **Directory Operations**
   - `ListDirectories()` - Get subfolders
   - `CreateDirectory()` - Create folder
   - `DirectoryExists()` - Check folder existence
   - `DeleteDirectory()` - Delete folder

3. **Account Management**
   - `GetAccounts()` - List registered accounts
   - `ShowAccountInformation()` - Show account details
   - `RegisterAccount()` - Register new account
   - `DeleteAccount()` - Delete account
   - `GetDescription()` - Connector description
   - `GetLogoAsBase64()` - Connector logo

### Graph API Module Capabilities

#### ✅ Fully Supported Operations
- ✅ **Download files** - `DownloadFile()`, `DownloadFileByPath()`
- ✅ **Upload files** - `UploadFile()`, `UploadLargeFile()` (with chunked upload)
- ✅ **Create folders** - `CreateFolder()` with conflict behavior support
- ✅ **List items** - `GetRootItems()`, `GetFolderItems()`, `GetItemsByPath()`
- ✅ **Get item metadata** - `GetDriveItem()`, `GetDriveItemByPath()`
- ✅ **Multi-drive support** - Can work with specific drives or default drive
- ✅ **Pagination** - Built-in pagination handling for large result sets
- ✅ **Error handling** - Comprehensive error handling via `SharePointGraphResponse`

#### ⚠️ Missing/Gap Operations
- ❌ **Delete file** - No direct delete method
- ❌ **Delete folder** - No direct delete method
- ❌ **Move file** - No direct move method
- ❌ **Copy file** - No direct copy method
- ❌ **File exists check** - No dedicated existence check (can use GetDriveItemByPath with error handling)
- ❌ **Folder exists check** - No dedicated existence check

---

## Gap Analysis

### Critical Gaps (Must Have)

1. **Delete Operations**
   - **Impact:** Required by `External File Storage Connector` interface
   - **Graph API Support:** ✅ Graph API supports DELETE on `/drives/{drive-id}/items/{item-id}`
   - **Implementation:** Need to add `DeleteItem()` method to Graph client
   - **Effort:** Low (1-2 hours)

2. **Existence Checks**
   - **Impact:** Required by interface (`FileExists()`, `DirectoryExists()`)
   - **Current Workaround:** Use `GetDriveItemByPath()` and check for errors
   - **Better Approach:** Add dedicated existence check methods that handle 404 gracefully
   - **Effort:** Low (1-2 hours)

### Nice to Have Gaps

3. **Copy Operation**
   - **Impact:** Medium - currently implemented via download+upload in REST connector
   - **Graph API Support:** ✅ Graph API supports native COPY operation
   - **Implementation:** Add `CopyItem()` method for more efficient copying
   - **Fallback:** Can use download+upload approach like REST connector
   - **Effort:** Medium (2-4 hours)

4. **Move Operation**
   - **Impact:** Medium - currently implemented via download+upload+delete
   - **Graph API Support:** ✅ Graph API supports PATCH to update parentReference
   - **Implementation:** Add `MoveItem()` method for more efficient moving
   - **Fallback:** Can use download+upload+delete approach
   - **Effort:** Medium (2-4 hours)

---

## Architecture Design

### Proposed Structure

```
External File Storage - SharePoint Connector/
├── src/
│   ├── ExtSharePointAccount.Table.al (shared)
│   ├── ExtSharePointAccountWizard.Page.al (shared)
│   ├── ExtSharePointAccount.Page.al (shared)
│   ├── ExtSharePointAuthType.Enum.al (shared - add "Graph API" value)
│   ├── ExtSharePointConnector.EnumExt.al (existing REST)
│   ├── ExtSharePointConnectorImpl.Codeunit.al (existing REST)
│   │
│   ├── graph/
│   │   ├── ExtSharePointGraphConnector.EnumExt.al (new)
│   │   └── ExtSharePointGraphConnectorImpl.Codeunit.al (new)
```

### Design Options

#### **Option 1: Separate Connector (Recommended)**
- **Approach:** Create a new enum value "SharePoint (Graph API)"
- **Pros:**
  - Users can choose REST vs Graph API
  - Allows A/B testing and gradual migration
  - Easier rollback if issues arise
  - Can maintain both during transition period
- **Cons:**
  - More code to maintain
  - Potential confusion for users

#### **Option 2: Replace Existing Connector**
- **Approach:** Swap out REST implementation with Graph implementation
- **Pros:**
  - Single connector to maintain
  - Cleaner architecture
  - Forces adoption of modern API
- **Cons:**
  - Risky - no easy rollback
  - Breaking change for existing users
  - All-or-nothing migration

#### **Option 3: Hybrid/Strategy Pattern**
- **Approach:** Single connector with internal strategy selection
- **Pros:**
  - Transparent to users
  - Can switch based on capability detection
- **Cons:**
  - Complex implementation
  - Harder to troubleshoot

**Recommendation:** Start with **Option 1** to minimize risk and allow gradual migration.

---

## Implementation Tasks

### Phase 1: Extend Graph API Module (Prerequisites) 🚨
**Goal:** Add missing CRITICAL operations to SharePoint Graph module

1. **🚨 Add Chunked Download Operations** (CRITICAL - BLOCKER)
   - [ ] Add `DownloadLargeFile(ItemId: Text; var FileOutStream: OutStream)` to `SharePointGraphClient`
   - [ ] Add `DownloadLargeFileByPath(FilePath: Text; var FileOutStream: OutStream)` to `SharePointGraphClient`
   - [ ] Implement chunked download using HTTP Range headers
   - [ ] Add `DownloadChunk()` helper to `SharePointGraphReqHelper`
   - [ ] Set Range header (e.g., `bytes=0-104857599`)
   - [ ] Handle 206 Partial Content responses
   - [ ] Use 100MB chunk size (safely under 150MB BC limit)
   - [ ] Get file size first using HEAD request or item metadata
   - [ ] Loop through chunks until entire file downloaded
   - [ ] Add error handling and retry logic

2. **Add Delete Operations** (Critical)
   - [ ] Add `DeleteItem(ItemId: Text)` to `SharePointGraphClient`
   - [ ] Add `DeleteItemByPath(ItemPath: Text)` to `SharePointGraphClient`
   - [ ] Implement in `SharePointGraphClientImpl`
   - [ ] Add corresponding URI builder methods
   - [ ] Add error handling and response parsing

3. **Add Existence Check Operations** (Critical)
   - [ ] Add `ItemExists(ItemId: Text): Boolean`
   - [ ] Add `ItemExistsByPath(ItemPath: Text): Boolean`
   - [ ] Handle 404 responses gracefully (return false, not error)

4. **Add Copy Operation** (Optional)
   - [ ] Add `CopyItem(ItemId: Text; TargetPath: Text)` 
   - [ ] Implement Graph API copy operation
   - [ ] Handle async copy status polling for large files

5. **Add Move Operation** (Optional)
   - [ ] Add `MoveItem(ItemId: Text; TargetPath: Text)`
   - [ ] Implement via PATCH parentReference

**Estimated Effort:** 2-3 days (increased due to chunked download complexity)

### Phase 2: Create Graph Connector Implementation
**Goal:** Implement `External File Storage Connector` interface using Graph API

1. **Create Core Files**
   - [ ] Create `ExtSharePointGraphConnector.EnumExt.al`
     ```al
     enumextension 4581 "Ext. SharePoint Graph Connector" extends "Ext. File Storage Connector"
     {
         value(4581; "SharePoint (Graph API)")
         {
             Caption = 'SharePoint (Graph API)';
             Implementation = "External File Storage Connector" = "Ext. SharePoint Graph Connector Impl";
         }
     }
     ```
   
   - [ ] Create `ExtSharePointGraphConnectorImpl.Codeunit.al`
     - Implement all interface methods
     - Initialize `SharePointGraphClient`
     - Map interface operations to Graph API calls

2. **Implement Interface Methods**

   **File Operations:**
   - [ ] `ListFiles()` → Use `GetItemsByPath()` + filter for files only (IsFolder = false)
   - [ ] `GetFile()` → 🚨 **Smart download based on size:**
     - Files ≤ 150MB: Use `DownloadFileByPath()` ✅
     - Files > 150MB: Use `DownloadLargeFileByPath()` ⚠️ (TO BE IMPLEMENTED - chunked)
     - Must check file size first via `GetDriveItemByPath()` to determine method
   - [ ] `CreateFile()` → Smart upload based on size:
     - Files < 4MB: Use `UploadFile()` ✅
     - Files ≥ 4MB: Use `UploadLargeFile()` ✅ (chunked, up to 250GB)
   - [ ] `CopyFile()` → Use `CopyItem()` (to be added) or fallback to download+upload
   - [ ] `MoveFile()` → Use `MoveItem()` (to be added) or fallback to download+upload+delete
   - [ ] `FileExists()` → Use `ItemExistsByPath()` (to be added) + check IsFolder = false
   - [ ] `DeleteFile()` → Use `DeleteItemByPath()` (to be added)

   **Directory Operations:**
   - [ ] `ListDirectories()` → Use `GetItemsByPath()` + filter for folders
   - [ ] `CreateDirectory()` → Use `CreateFolder()`
   - [ ] `DirectoryExists()` → Use `ItemExistsByPath()` + check IsFolder = true
   - [ ] `DeleteDirectory()` → Use `DeleteItemByPath()`

   **Account Management:**
   - [ ] `GetAccounts()` → Reuse existing `ExtSharePointAccount` table
   - [ ] `ShowAccountInformation()` → Reuse existing page
   - [ ] `RegisterAccount()` → Reuse existing wizard (may need minor updates)
   - [ ] `DeleteAccount()` → Reuse existing logic
   - [ ] `GetDescription()` → Return appropriate description
   - [ ] `GetLogoAsBase64()` → Reuse or create new logo

3. **Helper Methods**
   - [ ] `InitGraphClient()` - Initialize Graph client with auth
   - [ ] `InitPath()` - Combine base path with relative path
   - [ ] `SplitPath()` - Split path into parent and item name
   - [ ] `ShowError()` - Display Graph API errors
   - [ ] `FilterFiles()` - Helper to filter DriveItems for files only
   - [ ] `FilterFolders()` - Helper to filter DriveItems for folders only

**Estimated Effort:** 2-3 days

### Phase 3: Authentication Updates
**Goal:** Ensure Graph API authentication works properly

1. **Update Account Table** (if needed)
   - [ ] Review if `ExtSharePointAccount` needs any Graph-specific fields
   - [ ] Consider adding API type field to distinguish REST vs Graph accounts

2. **Update Authentication Setup**
   - [ ] Verify Graph API scopes are correct (`Sites.ReadWrite.All` or similar)
   - [ ] Ensure OAuth flow works for Graph endpoints
   - [ ] Test both Authorization Code and Client Credentials flows

3. **Update Wizard** (if needed)
   - [ ] Add option to select API type (REST vs Graph)
   - [ ] Update tooltips and help text
   - [ ] Validate Graph-specific requirements

**Estimated Effort:** 1 day

### Phase 4: Testing
**Goal:** Comprehensive testing of Graph implementation

1. **Unit Testing**
   - [ ] Test each interface method independently
   - [ ] Test error scenarios (404, 401, 403, etc.)
   - [ ] Test pagination with large folders
   - [ ] Test large file uploads (chunked)

2. **Integration Testing**
   - [ ] Test complete workflows (upload → list → download → delete)
   - [ ] Test with real SharePoint site
   - [ ] Test with different authentication types
   - [ ] Test multi-level folder structures

3. **Performance Testing**
   - [ ] Compare REST vs Graph performance
   - [ ] Test with large numbers of files
   - [ ] Test large file transfers

4. **Migration Testing**
   - [ ] Test side-by-side REST and Graph accounts
   - [ ] Document migration path for existing users

**Estimated Effort:** 2-3 days

---

## Chunked Upload/Download Analysis ✅

### Upload Implementation (Chunked) - **FULLY SUPPORTED**

The Graph API implementation has **excellent chunked upload support**:

1. **`UploadFile()`** - Simple upload for files < 4MB
   - Uses single PUT request
   - Best for small files
   - No session creation overhead

2. **`UploadLargeFile()`** - Chunked upload for files ≥ 4MB
   - **Chunk Size:** 4 MB (4,194,304 bytes)
   - **Minimum Chunk:** 320 KiB (327,680 bytes)
   - **Maximum File Size:** Up to 250GB
   - **Process:**
     1. Create upload session via `CreateUploadSession()`
     2. Split file into 4MB chunks
     3. Upload each chunk with `UploadChunk()` 
     4. Each chunk includes `Content-Range` header (e.g., `bytes 0-4194303/10485760`)
     5. Last chunk response contains the created item metadata
   - **Error Handling:** Each chunk is validated; if a chunk fails, the entire upload fails
   - **Resumable:** Uses upload session URL (though current implementation doesn't persist sessions)

### Download Implementation - ⚠️ **CRITICAL GAP: LACKS CHUNKED DOWNLOAD**

**Current Implementation:**

1. **`DownloadFile(ItemId)`** - Download by item ID
2. **`DownloadFileByPath(FilePath)`** - Download by path
3. **Current Behavior:**
   - Uses HTTP GET to `/content` endpoint
   - Returns InStream directly from HTTP response
   - ❌ **Limited to 150MB** due to BC HttpClient limitation
   - ❌ **Will fail** for files > 150MB

**🚨 PROBLEM: Business Central HttpClient Limitation**
```
Maximum response content size: 150 MB
Files above this size: FAIL to download
```

**✅ SOLUTION: Implement Chunked Download Using HTTP Range**

Graph API **DOES support** partial content requests via Range headers:

```al
// Proposed implementation for DownloadLargeFile()
procedure DownloadLargeFile(FilePath: Text; var FileOutStream: OutStream): Boolean
var
    ChunkSize: Integer;
    TotalBytes: Integer;
    BytesDownloaded: Integer;
    ChunkInStream: InStream;
    RangeHeader: Text;
begin
    ChunkSize := 100 * 1024 * 1024; // 100MB chunks (safely under 150MB limit)
    TotalBytes := GetFileSize(FilePath); // Get file size first
    BytesDownloaded := 0;
    
    while BytesDownloaded < TotalBytes do begin
        // Set Range header: "bytes=0-104857599" (0 to 100MB)
        RangeHeader := StrSubstNo('bytes=%1-%2', 
            BytesDownloaded, 
            BytesDownloaded + ChunkSize - 1);
        
        // Make GET request with Range header
        // Graph API returns 206 Partial Content
        DownloadChunk(FilePath, RangeHeader, ChunkInStream);
        
        // Write chunk to output stream
        CopyStream(FileOutStream, ChunkInStream);
        BytesDownloaded += ChunkSize;
    end;
end;
```

**Graph API Support:**
- ✅ Supports HTTP Range requests (per Microsoft Graph API spec)
- ✅ Returns 206 Partial Content
- ✅ Header: `Accept-Ranges: bytes`
- ✅ Allows downloading arbitrary byte ranges

**Implementation Details for SharePointGraphReqHelper:**

```al
/// <summary>
/// Downloads a chunk of file content using HTTP Range header.
/// </summary>
/// <param name="Endpoint">The endpoint to request.</param>
/// <param name="RangeStart">Starting byte position (0-based).</param>
/// <param name="RangeEnd">Ending byte position (inclusive).</param>
/// <param name="ChunkInStream">The stream to receive chunk content.</param>
/// <returns>True if the chunk was downloaded successfully.</returns>
procedure DownloadChunk(Endpoint: Text; RangeStart: Integer; RangeEnd: Integer; var ChunkInStream: InStream): Boolean
var
    HttpResponseMessage: Codeunit "Http Response Message";
    GraphOptionalParameters: Codeunit "Graph Optional Parameters";
    FinalEndpoint: Text;
begin
    // Set Range header: "bytes=0-104857599"
    GraphOptionalParameters.SetHeader('Range', StrSubstNo('bytes=%1-%2', RangeStart, RangeEnd));
    
    FinalEndpoint := PrepareEndpoint(Endpoint, GraphOptionalParameters);
    GraphClient.Get(FinalEndpoint, GraphOptionalParameters, HttpResponseMessage);
    
    // Should receive 206 Partial Content response
    if HttpResponseMessage.GetHttpStatusCode() <> 206 then
        exit(false);
    
    exit(ProcessStreamResponse(HttpResponseMessage, ChunkInStream));
end;

/// <summary>
/// Gets the size of a file without downloading it.
/// </summary>
procedure GetFileSize(ItemPath: Text): BigInteger
var
    GraphDriveItem: Record "SharePoint Graph Drive Item" temporary;
begin
    // Use HEAD request or GetDriveItemByPath to get metadata
    SharePointGraphClient.GetDriveItemByPath(ItemPath, GraphDriveItem);
    exit(GraphDriveItem.Size);
end;
```

**Key Considerations:**
- Use 100MB chunks to stay well under 150MB BC limit
- First get file size via metadata (GetDriveItemByPath)
- Loop through chunks: 0-100MB, 100MB-200MB, etc.
- Last chunk may be smaller than 100MB
- Assemble chunks into output stream sequentially
- Handle network errors per chunk (retry logic)

### Recommendation for Connector Implementation

**Upload (Already Supported):**
```al
procedure CreateFile(AccountId: Guid; Path: Text; Stream: InStream)
var
    FileSize: Integer;
begin
    FileSize := Stream.Length();
    
    // Use simple upload for small files (< 4MB)
    if FileSize < 4194304 then  // 4MB in bytes
        SharePointGraphClient.UploadFile(FolderPath, FileName, Stream, GraphDriveItem)
    else
        // Use chunked upload for large files (≥ 4MB)
        SharePointGraphClient.UploadLargeFile(FolderPath, FileName, Stream, GraphDriveItem);
end;
```

**Download (NEEDS IMPLEMENTATION):**
```al
procedure GetFile(AccountId: Guid; Path: Text; Stream: InStream)
var
    GraphDriveItem: Record "SharePoint Graph Drive Item" temporary;
    FileSize: BigInteger;
    TempBlob: Codeunit "Temp Blob";
    OutStream: OutStream;
begin
    // First, get file metadata to check size
    SharePointGraphClient.GetDriveItemByPath(Path, GraphDriveItem);
    FileSize := GraphDriveItem.Size;
    
    if FileSize <= 150000000 then begin  // 150MB limit
        // Safe to use simple download
        SharePointGraphClient.DownloadFileByPath(Path, Stream);
    end else begin
        // MUST use chunked download for files > 150MB
        TempBlob.CreateOutStream(OutStream);
        SharePointGraphClient.DownloadLargeFileByPath(Path, OutStream);  // ⚠️ TO BE IMPLEMENTED
        TempBlob.CreateInStream(Stream);
    end;
end;
```

**Benefits (After Implementation):**
- ✅ Upload: No file size limitations (up to 250GB)
- ✅ Efficient upload of large files with chunking
- ⚠️ Download: Will support files > 150MB (requires implementation)
- ✅ Better reliability with chunked operations

## Key Considerations

### 1. Path Handling Differences
- **REST API:** Uses server-relative URLs (e.g., `/sites/mysite/Shared Documents/folder/file.txt`)
- **Graph API:** Uses drive-relative paths (e.g., `folder/file.txt`)
- **Action:** Need path conversion logic in `InitPath()`

### 2. Pagination
- **REST API:** Limited pagination support
- **Graph API:** Built-in pagination with `@odata.nextLink`
- **Action:** Leverage Graph pagination for better performance with large folders

### 3. File Size Limits ✅ **CONFIRMED: FULLY SUPPORTED**
- **REST API:** Varies by implementation
- **Graph API:** 
  - **Simple upload:** Files up to 4MB via `UploadFile()` - uses PUT request
  - **Chunked upload:** Files > 4MB via `UploadLargeFile()` - supports up to **250GB**
    - Chunk size: **4MB** (recommended by Microsoft for optimal performance)
    - Chunk multiple: Must be multiples of **320 KiB** (327,680 bytes)
    - Uses upload session with `CreateUploadSession()` + `UploadChunk()` loop
  - **Download:** Direct streaming via `DownloadFile()` / `DownloadFileByPath()` - no size limits
- **Action:** ✅ Already implemented! Use `UploadFile()` for small files, `UploadLargeFile()` for large files
- **Note:** Download does NOT use chunking - Graph API handles this efficiently via HTTP streaming

### 4. Item Identification
- **REST API:** Uses server-relative paths
- **Graph API:** Uses item IDs or paths
- **Action:** May need to maintain path-to-ID mapping for some operations

### 5. Conflict Behavior
- **Graph API:** Built-in conflict behavior options (fail, replace, rename)
- **REST API:** Less flexible
- **Action:** Expose Graph's superior conflict handling where beneficial

### 6. Error Handling
- **REST API:** Custom error format
- **Graph API:** Standard Graph error format with detailed error codes
- **Action:** Implement comprehensive error mapping

---

## Authentication Differences

### Current (REST API)
```al
// Uses SharePoint-specific OAuth with scope:
Scopes.Add('00000003-0000-0ff1-ce00-000000000000/.default');
```

### Graph API
```al
// Requires Graph API scopes:
Scopes.Add('https://graph.microsoft.com/.default');
// Or specific permissions:
// Sites.ReadWrite.All, Files.ReadWrite.All
```

**Action:** Determine if account table needs separate scope configuration.

---

## Risk Assessment

### High Risk - BLOCKER
- 🚨 **Chunked download missing** - CRITICAL BLOCKER for files > 150MB
  - Impact: Cannot download files larger than 150MB without implementation
  - Mitigation: MUST implement chunked download in Phase 1 using HTTP Range headers
  - Complexity: Medium-High (requires Range request handling, chunk assembly)
  - Alternative: Limit connector to files ≤ 150MB (not recommended)
- ❌ **Delete operations missing** - MUST be implemented
  - Mitigation: Implement in Phase 1 before connector implementation

### Medium Risk
- ⚠️ **Path handling complexity** - Different path formats between APIs
  - Mitigation: Thorough testing with various path scenarios
- ⚠️ **Authentication changes** - Different scopes/permissions
  - Mitigation: Clear documentation and wizard guidance

### Low Risk
- ✅ **Feature parity** - Most operations have equivalent Graph APIs
- ✅ **Account management** - Can reuse existing infrastructure
- ✅ **Error handling** - Graph response structure is well-defined

---

## Success Criteria

1. ✅ All `External File Storage Connector` interface methods implemented
2. ✅ Delete operations added to Graph API module
3. ✅ All tests passing (unit, integration, performance)
4. ✅ Documentation complete (setup guide, migration guide)
5. ✅ Performance equivalent or better than REST implementation
6. ✅ Zero breaking changes for existing REST users

---

## Timeline Estimate

| Phase | Duration | Dependencies |
|-------|----------|--------------|
| Phase 1: Extend Graph Module | **2-3 days** | None - includes chunked download |
| Phase 2: Graph Connector | 2-3 days | Phase 1 complete |
| Phase 3: Authentication | 1 day | Phase 2 in progress |
| Phase 4: Testing | **3-4 days** | Phases 1-3 complete - extra time for large file testing |
| **Total** | **8-11 days** | - |

**Note:** Increased timeline due to chunked download implementation requirement

---

## Recommendation

**Proceed with implementation** using the following approach, **BUT** with critical caveat:

1. 🚨 **CRITICAL:** Implement chunked download FIRST - this is a blocker for files > 150MB
2. ✅ **Yes, Graph API module has potential** (requires significant additions)
3. ✅ **Implement Option 1** (Separate connector) for risk mitigation
4. ⚠️ **Start with Phase 1** to fill critical gaps, especially chunked download
5. ✅ **Create proof-of-concept** with file upload/download/delete before full implementation
6. ✅ **Plan for gradual migration** rather than forced switch

### Immediate Next Steps (REVISED)

1. 🚨 **PRIORITY 1:** Research and implement chunked download using HTTP Range headers
   - Test with Graph API to confirm Range support
   - Implement `DownloadChunk()` in `SharePointGraphReqHelper`
   - Implement `DownloadLargeFile()` in `SharePointGraphClient`
   - Test with files > 150MB
2. Add delete and existence check methods to `SharePointGraphClient`
3. Create proof-of-concept `ExtSharePointGraphConnectorImpl` with basic operations
4. **CRITICAL TEST:** Verify download of files > 150MB works correctly
5. Test PoC with real SharePoint environment with various file sizes
6. Decide on final architecture based on PoC results
7. Complete full implementation if PoC successful

### ⚠️ Alternative If Chunked Download Cannot Be Implemented

If HTTP Range-based chunked download proves impossible:
- **Option A:** Limit Graph connector to files ≤ 150MB (document limitation)
- **Option B:** Stay with REST connector (check if it has same limitation)
- **Option C:** Implement workaround using OneDrive API or different approach

---

## Open Questions

1. 🚨 **Can Graph API actually support HTTP Range requests for partial downloads?**
   - **CRITICAL:** Must verify this works before proceeding
   - Test: Make GET request with `Range: bytes=0-104857599` header
   - Expected: 206 Partial Content response
   - If NO: Project is blocked for files > 150MB

2. **Does REST connector have the same 150MB download limitation?**
   - Need to verify if current REST implementation can download files > 150MB
   - If YES: Both connectors need chunked download
   - If NO: Understand how REST avoids the limitation

3. **Should we support both REST and Graph simultaneously?**
   - Recommendation: Yes, at least initially

4. **How should we handle accounts created with REST connector?**
   - Recommendation: Allow users to migrate or create new Graph-based accounts

5. **Should Graph connector be the default for new accounts?**
   - Recommendation: Only after chunked download is verified working

6. **Do we need a migration tool?**
   - Recommendation: Not necessary - accounts are configuration, not data

7. **Should we eventually deprecate REST connector?**
   - Recommendation: Only if Graph connector fully supports all file sizes (6-12 months)

---

## Appendix: API Mapping Reference

| Interface Method | REST API Method | Graph API Method | Notes |
|------------------|----------------|------------------|-------|
| `ListFiles` | `GetFolderFilesByServerRelativeUrl` | `GetItemsByPath` + filter | Filter `IsFolder=false` |
| `GetFile` | `DownloadFileContentByServerRelativeUrl` | `DownloadFileByPath` | ✅ Direct mapping |
| `CreateFile` | `AddFileToFolder` | `UploadFile` or `UploadLargeFile` | Size-based selection |
| `CopyFile` | Download + Upload | `CopyItem` (new) | Graph more efficient |
| `MoveFile` | Download + Upload + Delete | `MoveItem` (new) | Graph more efficient |
| `FileExists` | `GetFolderFiles` + filter | `ItemExistsByPath` (new) | Better performance |
| `DeleteFile` | `DeleteFileByServerRelativeUrl` | `DeleteItemByPath` (new) | ⚠️ Needs implementation |
| `ListDirectories` | `GetSubFoldersByServerRelativeUrl` | `GetItemsByPath` + filter | Filter `IsFolder=true` |
| `CreateDirectory` | `CreateFolder` | `CreateFolder` | ✅ Direct mapping |
| `DirectoryExists` | `FolderExistsByServerRelativeUrl` | `ItemExistsByPath` (new) | Better performance |
| `DeleteDirectory` | `DeleteFolderByServerRelativeUrl` | `DeleteItemByPath` (new) | ⚠️ Needs implementation |

**Legend:**
- ✅ = Already available in Graph module
- ⚠️ = Needs to be implemented
- (new) = New method to add

---

**Document Version:** 1.0  
**Last Updated:** October 29, 2025
