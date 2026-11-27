# Code AWS đã thêm theo List.txt

## 1. Test 2000 user

Không có code thêm.

## 2. Host AWS thay IP tĩnh

Chỉ nhận URL `https://<bucket>.s3.<region>.amazonaws.com/`. IP và host khác không bật AWS.

`PrototypeRacing/Plugins/ChunkDownloaderCustom/Source/ChunkDownloaderCustom/Private/ChunkDownloaderSubsystem.cpp`

```cpp
	bool IsAwsCdnUrl(const FString& Url)
	{
		// Regional S3 roots supplied by CI; no IP, credentials or arbitrary HTTPS host.
		const FRegexPattern Pattern(TEXT("^https://[a-z0-9][a-z0-9.-]{1,61}[a-z0-9]\\.s3\\.[a-z0-9-]+\\.amazonaws\\.com/"));
		FRegexMatcher Matcher(Pattern, Url);
		return Matcher.FindNext();
	}
```

AWS được bật khi `CdnBaseUrl` khớp hàm trên.

`PrototypeRacing/Plugins/ChunkDownloaderCustom/Source/ChunkDownloaderCustom/Private/ChunkDownloaderSubsystem.cpp`

```cpp
bool UChunkDownloaderSubsystem::IsAwsCdnEnabled() const
{
	return IsAwsCdnUrl(CdnBaseUrl);
}
```

Cờ test chỉ đọc khi URL đã là S3. Bản Shipping thấy cờ này thì dừng. 

`PrototypeRacing/Plugins/ChunkDownloaderCustom/Source/ChunkDownloaderCustom/Private/ChunkDownloaderSubsystem.cpp`

```cpp
	PrimaryCdnBaseUrl = CdnBaseUrl;
	bAwsLocalTestMode = false;
	if (IsAwsCdnEnabled())
	{
		GConfig->GetBool(TEXT("ChunkDownloaderPrototype"), TEXT("bAwsLocalTestMode"), bAwsLocalTestMode, GGameIni);
#if UE_BUILD_SHIPPING
		if (bAwsLocalTestMode)
		{
			SetError(NSLOCTEXT("ChunkDownloaderPrototype", "AwsLocalShipping", "AWS local test không được dùng trong bản Shipping."));
			return false;
		}
#endif
	}

	const bool bIsVersionedBuild = ContentBuildId.StartsWith(TEXT("Racing_FullCatalog_"), ESearchCase::CaseSensitive);
	const bool bIsLocalMock = CdnBaseUrl.StartsWith(TEXT("http://127.0.0.1"), ESearchCase::IgnoreCase)
		|| CdnBaseUrl.StartsWith(TEXT("http://localhost"), ESearchCase::IgnoreCase)
		|| CdnBaseUrl.StartsWith(TEXT("http://192.168."), ESearchCase::IgnoreCase);

	const bool bValidCdnPrefix = bIsLocalMock
		|| IsAllowedClickStudioCdnBaseUrl(CdnBaseUrl) || IsAwsCdnEnabled();
	if (bIsVersionedBuild && !bValidCdnPrefix)
	{
		SetError(NSLOCTEXT("ChunkDownloaderPrototype", "V6AuthorityMismatch", "Cấu hình máy chủ dữ liệu không hợp lệ."));
		return false;
	}
```

Bản Development và cờ test đang bật thì bỏ kiểm RS256. SHA-256 từng segment vẫn kiểm khi ghép file.

`PrototypeRacing/Plugins/ChunkDownloaderCustom/Source/ChunkDownloaderCustom/Private/ChunkDownloaderSubsystem.cpp`

```cpp
	const FString Signature = RootObject->GetStringField(TEXT("signature"));
	const FString KeyId = RootObject->GetStringField(TEXT("keyId"));

	const bool bIsLocalMockCdn = CdnBaseUrl.StartsWith(TEXT("http://127.0.0.1"), ESearchCase::IgnoreCase)
		|| CdnBaseUrl.StartsWith(TEXT("http://localhost"), ESearchCase::IgnoreCase)
		|| CdnBaseUrl.StartsWith(TEXT("http://192.168."), ESearchCase::IgnoreCase)
#if !UE_BUILD_SHIPPING
		|| (IsAwsCdnEnabled() && bAwsLocalTestMode)
#endif
		;

	if (!bIsLocalMockCdn)
	{
		if (Signature.IsEmpty() || KeyId.IsEmpty())
		{
			UE_LOG(LogChunkDownloaderSubsystem, Error, TEXT("Manifest V2 missing signature or keyId."));
			return false;
		}
		if (!VerifyV6ManifestSignature(RootObject, Signature, KeyId))
		{
			UE_LOG(LogChunkDownloaderSubsystem, Error, TEXT("Manifest V2 RS256 signature verification failed. KeyId=%s"), *KeyId);
			return false;
		}
	}
	else
	{
		UE_LOG(LogChunkDownloaderSubsystem, Log, TEXT("Local Mock CDN detected (%s): RS256 signature check bypassed for offline developer test."), *CdnBaseUrl);
	}
```

Host S3 duy nhất mà build local chấp nhận.

`tools/ci_local_package_and_segment.py`

```python
AWS_ROOT = "https://vnracing-cdn.s3.ap-southeast-1.amazonaws.com"
```

Có `public_base` thì ghi URL S3 và `bAwsLocalTestMode=True`. Không có thì URL vẫn là MinIO.

`tools/ci_local_package_and_segment.py`

```python
    if public_base:
        cdn_base_url = f"{public_base.rstrip('/')}/{release_prefix}/"
        pipeline_endpoint = public_base.rstrip("/")
    else:
        cdn_base_url = f"{minio_endpoint.rstrip('/')}/{target_bucket}/{release_prefix}/"
        pipeline_endpoint = minio_endpoint

    print(f"[CONFIG] ContentBuildId:   {content_build_id}")
    print(f"[CONFIG] CdnReleasePrefix: {release_prefix}")
    print(f"[CONFIG] CdnBaseUrl:       {cdn_base_url}")

    replacements = {
        "ContentBuildId": content_build_id,
        "CdnReleasePrefix": release_prefix,
        "CdnBaseUrl": f'"{cdn_base_url}"',
        "ManifestName": f"BuildManifest-{platform}.json",
        "PipelineP4Changelist": str(changelist),
        "PipelineJenkinsBuildNumber": str(build_num),
        "PipelineCdnEndpoint": f'"{pipeline_endpoint}"',
    }
    if public_base:
        replacements["bAwsLocalTestMode"] = "True"
```

`--cdn-target aws` đọc `aws.local.ini`. `public_base_url` phải đúng `AWS_ROOT`.

`tools/ci_local_package_and_segment.py`

```python
    aws_target = None
    public_base = None
    if args.cdn_target == "aws":
        # Local AWS uses the same existing local signing pair as the MinIO build.
        import boto3
        try:
            aws_target = load_aws_target(WORKSPACE_ROOT / "tools" / "aws.local.ini")
        except (FileNotFoundError, KeyError, ValueError) as err:
            print(f"[ERROR] AWS target is missing from tools/aws.local.ini ({err})")
            return 2
        public_base = aws_target["public_base_url"]
        if public_base.rstrip("/") != AWS_ROOT:
            print(f"[ERROR] AWS public_base_url must match {AWS_ROOT}.")
            return 2
```

## 3. Chỉ tải chunk đổi hash

Journal client không sửa. Phần thêm là upload S3 bỏ object cùng kích thước và cùng MD5.

S3 bỏ qua file đã có cùng kích thước và ETag trùng MD5. ETag có dấu `-` thì tải lại. Không có `s3_client` thì vẫn HEAD MinIO.

`tools/ci_local_package_and_segment.py`

```python
    base_url = f"{minio_endpoint.rstrip('/')}/{release_prefix}" if s3_client else f"{minio_endpoint.rstrip('/')}/{target_bucket}/{release_prefix}"

    def object_is_present(fpath: Path) -> bool:
        if s3_client is None:
            return head_exists(f"{base_url}/{fpath.name}")
        try:
            head = s3_client.head_object(Bucket=target_bucket, Key=f"{release_prefix}/{fpath.name}")
            if int(head.get("ContentLength", -1)) != fpath.stat().st_size:
                return False
            etag = str(head.get("ETag", "")).strip('"')
            if "-" in etag:
                return False
            digest = hashlib.md5()
            with fpath.open("rb") as handle:
                for chunk in iter(lambda: handle.read(1024 * 1024), b""):
                    digest.update(chunk)
            return etag == digest.hexdigest()
        except Exception:
            return False

    def upload_one(fpath: Path) -> bool:
        if s3_client is None:
            return put_with_retry(f"{base_url}/{fpath.name}", fpath)
        body = fpath.read_bytes()
        key = f"{release_prefix}/{fpath.name}"
        for attempt in range(1, 6):
            try:
                s3_client.put_object(Bucket=target_bucket, Key=key, Body=body, ContentType="application/octet-stream")
                return True
            except Exception as err:
                wait = min(2 ** attempt, 30)
                print(f"    [RETRY {attempt}/5] {fpath.name} in {wait}s ({err})", flush=True)
                time.sleep(wait)
        return False
```

## 4. Admin tạm ngừng CDN

Ghi `cdn-access.json` ở gốc bucket. Pause ghi `reason` là `updating`.

`tools/ci_local_package_and_segment.py`

```python
def publish_cdn_access(paused: bool, *, s3_client=None, bucket: str | None = None, http_base: str | None = None) -> bool:
    """Write cdn-access.json beside the release prefix. Clients stop when paused is true."""
    body = b'{"paused":true,"reason":"updating"}' if paused else b'{"paused":false,"reason":""}'
    try:
        if s3_client is not None:
            s3_client.put_object(Bucket=bucket, Key="cdn-access.json", Body=body, ContentType="application/json")
            print(f"[CDN ACCESS] paused={str(paused).lower()} s3://{bucket}/cdn-access.json")
            return True
        url = f"{http_base.rstrip('/')}/cdn-access.json"
        request = urllib.request.Request(url, data=body, method="PUT", headers={"Content-Type": "application/json"})
        with urllib.request.urlopen(request, timeout=60) as response:
            if response.status not in (200, 201, 204):
                print(f"[ERROR] cdn-access.json returned HTTP {response.status}")
                return False
        print(f"[CDN ACCESS] paused={str(paused).lower()} {url}")
        return True
    except Exception as err:
        print(f"[ERROR] cdn-access.json ({err})")
        return False
```

`--cdn-access pause` hoặc `resume` ghi file đó rồi thoát, không cook.

`tools/ci_local_package_and_segment.py`

```python
    parser.add_argument("--cdn-target", default="minio", choices=["minio", "aws"], help="minio is the internal upload. aws uses the same publisher with tools/aws.local.ini.")
    parser.add_argument("--cdn-access", choices=["pause", "resume"], help="Write cdn-access.json and exit. pause stops clients; resume lets them continue.")
    parser.add_argument("--target-bucket", default="cdn-admin", choices=["cdn-admin", "cdn-dev2"], help="MinIO target bucket on sproxy (default: cdn-admin)")
    parser.add_argument("--minio-endpoint", default=DEFAULT_MINIO_ENDPOINT, help=f"MinIO S3 endpoint (default: {DEFAULT_MINIO_ENDPOINT})")
    parser.add_argument("--stage-dir", type=Path, default=None, help="Override path to Staged directory")
    parser.add_argument("--output-dir", type=Path, default=None, help="Output directory for sliced segments")
    parser.add_argument("--changelist", type=int, default=None, help="Changelist number (default: auto-detected from p4)")
    parser.add_argument("--build-number", type=int, default=1, help="Local build iteration number (default: 1)")
    parser.add_argument("--ue-dir", default=None, help="Engine root that contains Engine\\Build\\BatchFiles\\RunUAT.bat. Overrides UE5_DIR and the registry.")
    parser.add_argument("--skip-gate", action="store_true", help="Skip pre-cook chunk quality gate")
    parser.add_argument("--skip-minio", action="store_true", help="Skip uploading to MinIO server")
    parser.add_argument("--install", action="store_true", help="Auto-install APK and Core OBB to connected Android device via ADB")
    parser.add_argument("--clear-data", action="store_true",
                        help="With --install, also wipe app data (pm clear). This deletes files/PakCache, "
                             "so every downloaded DLC chunk must be fetched again from the CDN.")

    args = parser.parse_args()
    if args.cdn_access:
        paused = args.cdn_access == "pause"
        if args.cdn_target == "aws":
            import boto3
            from botocore.config import Config
            target = load_aws_target(WORKSPACE_ROOT / "tools" / "aws.local.ini")
            client = boto3.client(
                "s3",
                region_name=target["region"],
                aws_access_key_id=target["aws_access_key_id"],
                aws_secret_access_key=target["aws_secret_access_key"],
                config=Config(signature_version="s3v4"),
            )
            return 0 if publish_cdn_access(paused, s3_client=client, bucket=target["bucket"]) else 3
        http_base = f"{args.minio_endpoint.rstrip('/')}/{args.target_bucket}"
        return 0 if publish_cdn_access(paused, http_base=http_base) else 3
```

Upload ghi pause trước, đẩy file, rồi ghi resume.

`tools/ci_local_package_and_segment.py`

```python
    if aws_target is not None or not args.skip_minio:
        target_name = "AWS" if s3_client is not None else "MinIO"
        print_step(5, f"Resilient Publishing to {target_name}")
        if not publish_cdn_access(True, **access_options):
            return 3
        if not resilient_publish_to_minio(output_dir, publish_endpoint, publish_bucket, release_prefix, s3_client=s3_client):
            return 3
        if not publish_cdn_access(False, **access_options):
            return 3
```

URL S3 thì kiểm pause trước khi tải. URL khác tải manifest ngay.

`PrototypeRacing/Plugins/ChunkDownloaderCustom/Source/ChunkDownloaderCustom/Private/ChunkDownloaderSubsystem.cpp`

```cpp
	if (IsAwsCdnEnabled())
	{
		BeginCdnAccessCheck();
		return;
	}

	// Internal CDN keeps the depot path: fetch the manifest immediately.
	HandleBuildUpdated(true);
}

void UChunkDownloaderSubsystem::BeginCdnAccessCheck()
{
	if (bIsShuttingDown || !IsAwsCdnEnabled())
	{
		return;
	}

	if (UWorld* World = GetWorld())
	{
		World->GetTimerManager().ClearTimer(ManifestTimeoutTimerHandle);
	}
	bAwsBootstrapPending = true;
	bAwsAccessBlocked = true;
	if (!AwsAccessTicker.IsValid())
	{
		AwsAccessTicker = FTSTicker::GetCoreTicker().AddTicker(FTickerDelegate::CreateWeakLambda(this, [this](float)
		{
			PollAwsAccess();
			return !bIsShuttingDown;
		}), 1.0f);
	}
	PollAwsAccess();
}
```

Mỗi 2 giây đọc file pause. `paused:true` thì hủy segment đang tải và hiện `reason`.

`PrototypeRacing/Plugins/ChunkDownloaderCustom/Source/ChunkDownloaderCustom/Private/ChunkDownloaderSubsystem.cpp`

```cpp
namespace
{
	FString BuildAwsCdnAccessUrl(const FString& CdnBaseUrl)
	{
		FString Root = CdnBaseUrl;
		while (Root.EndsWith(TEXT("/")))
		{
			Root.LeftChopInline(1);
		}
		const FString HostMark = TEXT(".amazonaws.com");
		const int32 HostEnd = Root.Find(HostMark, ESearchCase::IgnoreCase, ESearchDir::FromEnd);
		if (HostEnd == INDEX_NONE)
		{
			return FString();
		}
		return Root.Left(HostEnd + HostMark.Len()) + TEXT("/cdn-access.json?t=") + LexToString(FPlatformTime::Cycles64());
	}
}

void UChunkDownloaderSubsystem::StopAwsDownloads()
{
	TArray<TSharedPtr<IHttpRequest, ESPMode::ThreadSafe>> Requests;
	ActiveSegmentRequests.GetKeys(Requests);
	for (const TSharedPtr<IHttpRequest, ESPMode::ThreadSafe>& Request : Requests)
	{
		if (Request.IsValid())
		{
			Request->CancelRequest();
		}
	}
	if (!Requests.IsEmpty())
	{
		Journal.SaveAtomic();
	}
	SetDownloadScreenWakeLockEnabled(false);
}

void UChunkDownloaderSubsystem::PollAwsAccess()
{
	if (bIsShuttingDown || !IsAwsCdnEnabled())
	{
		return;
	}

	const double Now = FPlatformTime::Seconds();
	if (bAwsAccessBlocked && AwsPauseStarted > 0.0 && !ActiveSegmentRequests.IsEmpty())
	{
		StopAwsDownloads();
	}
	if (AwsAccessRequest.IsValid() || Now < AwsNextAccessCheck)
	{
		return;
	}

	const FString AccessUrl = BuildAwsCdnAccessUrl(CdnBaseUrl);
	if (AccessUrl.IsEmpty())
	{
		UE_LOG(LogChunkDownloaderSubsystem, Warning, TEXT("CDN access URL could not be built from %s"), *CdnBaseUrl);
		AwsNextAccessCheck = Now + 2.0;
		if (AwsPauseStarted <= 0.0 && bAwsBootstrapPending)
		{
			bAwsAccessBlocked = false;
			bAwsBootstrapPending = false;
			HandleBuildUpdated(true);
		}
		return;
	}

	AwsNextAccessCheck = Now + 2.0;
	AwsAccessRequest = FHttpModule::Get().CreateRequest();
	AwsAccessRequest->SetURL(AccessUrl);
	AwsAccessRequest->SetVerb(TEXT("GET"));
	AwsAccessRequest->SetHeader(TEXT("Cache-Control"), TEXT("no-cache"));
	AwsAccessRequest->SetHeader(TEXT("Pragma"), TEXT("no-cache"));
	AwsAccessRequest->SetTimeout(8.f);
	AwsAccessRequest->OnProcessRequestComplete().BindWeakLambda(this, [this, AccessUrl](FHttpRequestPtr, FHttpResponsePtr Response, bool bSuccess)
	{
		AwsAccessRequest.Reset();
		TSharedPtr<FJsonObject> State;
		bool bPaused = false;
		const int32 ResponseCode = Response.IsValid() ? Response->GetResponseCode() : 0;
		const bool bValid = bSuccess && Response.IsValid() && EHttpResponseCodes::IsOk(ResponseCode)
			&& FJsonSerializer::Deserialize(TJsonReaderFactory<>::Create(Response->GetContentAsString()), State)
			&& State.IsValid() && State->TryGetBoolField(TEXT("paused"), bPaused);
		UE_LOG(LogChunkDownloaderSubsystem, Log, TEXT("CDN access %s http=%d valid=%d paused=%d"), *AccessUrl, ResponseCode, bValid ? 1 : 0, bPaused ? 1 : 0);
		const bool bWasBlocked = bAwsAccessBlocked;
		if (!bValid)
		{
			if (AwsPauseStarted > 0.0)
			{
				bAwsAccessBlocked = true;
				BroadcastProgress();
				return;
			}
			bAwsAccessBlocked = false;
			if (bAwsBootstrapPending)
			{
				bAwsBootstrapPending = false;
				HandleBuildUpdated(true);
			}
			else if (bWasBlocked)
			{
				PumpSegmentDownloadQueue();
			}
			BroadcastProgress();
			return;
		}

		bAwsAccessBlocked = bPaused;
		if (bPaused)
		{
			if (AwsPauseStarted == 0.0)
			{
				AwsPauseStarted = FPlatformTime::Seconds();
			}
			FString Reason;
			State->TryGetStringField(TEXT("reason"), Reason);
			CurrentProgress.SidecarRetryStatus = Reason.IsEmpty()
				? TEXT("CDN đang tạm ngừng để cập nhật.")
				: Reason;
			StopAwsDownloads();
		}
		else
		{
			const bool bResume = bWasBlocked || AwsPauseStarted > 0.0;
			AwsPauseStarted = 0.0;
			CurrentProgress.SidecarRetryStatus.Reset();
			if (bAwsBootstrapPending)
			{
				bAwsBootstrapPending = false;
				HandleBuildUpdated(true);
			}
			else if (bResume)
			{
				SetDownloadScreenWakeLockEnabled(CurrentProgress.State == EChunkDownloaderPrototypeState::DownloadingChunks);
				PumpSegmentDownloadQueue();
			}
		}
		BroadcastProgress();
	});
	if (!AwsAccessRequest->ProcessRequest())
	{
		UE_LOG(LogChunkDownloaderSubsystem, Warning, TEXT("CDN access request failed to start: %s"), *AccessUrl);
		AwsAccessRequest.Reset();
		if (AwsPauseStarted <= 0.0 && bAwsBootstrapPending)
		{
			bAwsAccessBlocked = false;
			bAwsBootstrapPending = false;
			HandleBuildUpdated(true);
		}
	}
}
```

Đang pause thì không mở lượt tải mới.

`PrototypeRacing/Plugins/ChunkDownloaderCustom/Source/ChunkDownloaderCustom/Private/ChunkDownloaderSubsystem.cpp`

```cpp
void UChunkDownloaderSubsystem::PumpSegmentDownloadQueue()
{
	if (IsAwsCdnEnabled() && bAwsAccessBlocked)
	{
		return;
	}
```

Giữa vòng tải, vừa bị pause thì dừng, không lấy segment tiếp.

`PrototypeRacing/Plugins/ChunkDownloaderCustom/Source/ChunkDownloaderCustom/Private/ChunkDownloaderSubsystem.cpp`

```cpp
	while (ActiveSegmentRequests.Num() < EffectiveConcurrency && !SegmentDownloadQueue.IsEmpty())
	{
		if (IsAwsCdnEnabled() && bAwsAccessBlocked)
		{
			return;
		}
```

Segment bị hủy vì pause được xếp lại hàng, không tính là mất mạng.

`PrototypeRacing/Plugins/ChunkDownloaderCustom/Source/ChunkDownloaderCustom/Private/ChunkDownloaderSubsystem.cpp`

```cpp
				const int32 ResponseCode = Response.IsValid() ? Response->GetResponseCode() : 0;
				const int64 FinalPartSize = IFileManager::Get().FileSize(*PartPath);
				const bool bHttpSuccessCode = (ResponseCode == 200 || ResponseCode == 206);
				const bool bHttpOk = bRequestSuccess && Response.IsValid() && bHttpSuccessCode && (FinalPartSize == ExpectedLength);
				if (Subsystem->IsAwsCdnEnabled() && Subsystem->bAwsAccessBlocked && !bHttpOk)
				{
					Subsystem->SegmentDownloadQueue.AddUnique(ObjectKey);
					Subsystem->ActiveSegmentLastActivityTime.Remove(ObjectKey);
					Subsystem->ActiveSegmentLastBytesReceived.Remove(ObjectKey);
					return;
				}
```

## 5. Đợi chunk rồi mới chạy tiếp

Ghi cùng 11 chunk vào Catalog, DeviceProof và RuntimeRequired.

`tools/ci_local_package_and_segment.py`

```python
CI_ROSTER_CHUNK_IDS = (1001, 1005, 1010, 1011, 1012, 1013, 1014, 1015, 1016, 1017, 9999)
BOOT_PROBES = (
    '+RuntimeProbes=(ProbeId="1005_SHARED",ChunkId=1005,AssetPath="/Game/Asset_Artist_No_Build/Meshes/Cars/Car_VFX/SM_LuxA_Body_Basic.SM_LuxA_Body_Basic",ExpectedClass="StaticMesh",ProbeKind="LoadObject",bRequired=True)',
    '+RuntimeProbes=(ProbeId="1010_SONTRA",ChunkId=1010,AssetPath="/Game/Maps/SonTra/ST_New/ST_Circuit.ST_Circuit",ExpectedClass="World",ProbeKind="LoadObject",bRequired=True)',
    '+RuntimeProbes=(ProbeId="1011_DOWNTOWN",ChunkId=1011,AssetPath="/Game/Maps/Downtown/DT_New/DT_Circuit.DT_Circuit",ExpectedClass="World",ProbeKind="LoadObject",bRequired=True)',
    '+RuntimeProbes=(ProbeId="1012_HUE_DAINOI",ChunkId=1012,AssetPath="/Game/Maps/HUE/DAINOI/Racing_Hue_Circuit.Racing_Hue_Circuit",ExpectedClass="World",ProbeKind="LoadObject",bRequired=True)',
    '+RuntimeProbes=(ProbeId="1013_NHS",ChunkId=1013,AssetPath="/Game/Maps/NHS/NHS_New/NHS_Circuit.NHS_Circuit",ExpectedClass="World",ProbeKind="LoadObject",bRequired=True)',
    '+RuntimeProbes=(ProbeId="1014_DONGGIANG",ChunkId=1014,AssetPath="/Game/Maps/DongGiang/DongGiang_New/DG_Circuit.DG_Circuit",ExpectedClass="World",ProbeKind="LoadObject",bRequired=True)',
    '+RuntimeProbes=(ProbeId="1015_HOIAN",ChunkId=1015,AssetPath="/Game/Technical_Art/Levels/HAn_Replacement/LV_HAn_Circuit_replace.LV_HAn_Circuit_replace",ExpectedClass="World",ProbeKind="LoadObject",bRequired=True)',
    '+RuntimeProbes=(ProbeId="1016_QUOCHOC",ChunkId=1016,AssetPath="/Game/Maps/HUE/QUOCHOC/Racing_QuocHoc_Circuit.Racing_QuocHoc_Circuit",ExpectedClass="World",ProbeKind="LoadObject",bRequired=True)',
    '+RuntimeProbes=(ProbeId="1017_CVTT",ChunkId=1017,AssetPath="/Game/Maps/HUE/CongVienTT/Racing_Hue_CVTT_Circuit.Racing_Hue_CVTT_Circuit",ExpectedClass="World",ProbeKind="LoadObject",bRequired=True)',
    '+RuntimeProbes=(ProbeId="9999_SHARED",ChunkId=9999,AssetPath="/Game/Asset_Artist_No_Build/Materials/3/M_Transparent.M_Transparent",ExpectedClass="Material",ProbeKind="LoadObject",bRequired=True)',
)


def ensure_boot_probes(text: str) -> str:
    newline = "\r\n" if "\r\n" in text else "\n"
    missing = [line for line in BOOT_PROBES if line.split("ProbeId=", 1)[1].split(",", 1)[0] not in text]
    if not missing:
        return text
    addition = newline.join(missing) + newline
    marker = "+RuntimeProbes="
    last = text.rfind(marker)
    if last == -1:
        anchor = "[ChunkDownloaderPrototype]"
        if anchor not in text:
            return text
        return text.replace(
            anchor,
            anchor + newline + "!RuntimeProbes=ClearArray" + newline + addition,
            1,
        )
    line_end = text.find("\n", last)
    if line_end == -1:
        return text + newline + addition
    return text[: line_end + 1] + addition + text[line_end + 1 :]


def inject_boot_roster(project_root: Path, ini_paths: list[Path]) -> bool:
    """Bake the CI/CD roster. Catalog, proof and required are the same list."""
    del project_root
    roster_ids = list(CI_ROSTER_CHUNK_IDS)
    print("[CONFIG] CI roster: " + ", ".join(str(chunk_id) for chunk_id in roster_ids))
    for ini_path in ini_paths:
        text = ini_path.read_text(encoding="utf-8")
        try:
            text = write_chunk_id_array(text, "CatalogChunkIds", roster_ids)
            text = write_chunk_id_array(text, "DeviceProofChunkIds", roster_ids)
            text = write_chunk_id_array(text, "RuntimeRequiredChunkIds", roster_ids)
        except ValueError as err:
            print(f"[ERROR] {ini_path}: {err}")
            return False
        if ini_path.name == "DefaultGame.ini":
            text = ensure_boot_probes(text)
        ini_path.write_text(text, encoding="utf-8")
        print(f"[CONFIG] Boot roster written to {ini_path}")
    return True
```

Sau boot, URL S3 đi qua guard. URL khác vẫn `OpenLevel`.

`PrototypeRacing/Plugins/ChunkDownloaderCustom/Source/ChunkDownloaderCustom/Private/ChunkDownloaderWidget.cpp`

```cpp
	if (UGameInstance* GameInstance = GetGameInstance())
	{
		if (UChunkDownloaderSubsystem* Transport = GameInstance->GetSubsystem<UChunkDownloaderSubsystem>();
			Transport && Transport->IsAwsCdnEnabled())
		{
			if (UChunkDownloadManagerSubsystem* ChunkManager = GameInstance->GetSubsystem<UChunkDownloadManagerSubsystem>())
			{
				ChunkManager->RequestLevelTravelWithChunkGuard(GarageWorkshopMapName);
				return;
			}
		}
	}
	UGameplayStatics::OpenLevel(this, GarageWorkshopMapName);
```

Một chunk trong DeviceProof chưa mount thì chưa mở map.

`PrototypeRacing/Plugins/ChunkDownloaderCustom/Source/ChunkDownloaderCustom/Private/ChunkDownloaderWidget.cpp`

```cpp
bool UChunkDownloaderWidget::AllRequiredChunksMounted() const

{

	const UChunkDownloaderSubsystem* Subsystem = GetPatchSubsystem();

	if (!Subsystem)

	{

		return false;

	}



	const TArray<int32> RequiredChunkIds = Subsystem->GetDeviceProofChunkIds();

	if (RequiredChunkIds.IsEmpty())

	{

		return false;

	}



	for (const int32 ChunkId : RequiredChunkIds)

	{

		if (!Subsystem->IsChunkMounted(ChunkId))

		{

			return false;

		}

	}



	return true;
```

URL S3 chỉ mở map khi chunk của map và chunk cha đã mount. Chưa đủ thì tải tiếp.

`PrototypeRacing/Plugins/ChunkDownloaderCustom/Source/ChunkDownloaderCustom/Private/ChunkDownloadManagerSubsystem.cpp`

```cpp
bool UChunkDownloadManagerSubsystem::RequestLevelTravelWithChunkGuard(FName TargetMapName, int32 RequiredChunkId, bool bPromptModalIfUnmounted, bool bServerTravel)
{
	if (TargetMapName.IsNone())
	{
		UE_LOG(LogChunkManager, Warning, TEXT("RequestLevelTravelWithChunkGuard: Invalid TargetMapName (None)."));
		return false;
	}

	int32 ResolvedChunkId = RequiredChunkId;
	if (ResolvedChunkId < 0)
	{
		ResolvedChunkId = GetChunkIdForMap(TargetMapName);
	}

	if (const UChunkDownloaderSubsystem* Transport = GetTransportSubsystem(); Transport && Transport->IsAwsCdnEnabled() && ResolvedChunkId > 0)
	{
		TArray<int32> Closure;
		if (GetAwsChunkClosure({ResolvedChunkId}, Closure))
		{
			if (AreAwsChunksReady({ResolvedChunkId}))
			{
				UE_LOG(LogChunkManager, Log, TEXT("RequestLevelTravelWithChunkGuard: Chunk %d and its parents are mounted. Travelling to '%s'."),
					ResolvedChunkId, *TargetMapName.ToString());
				TravelToResolvedMap(TargetMapName, bServerTravel);
				return true;
			}

			PendingTravelMapName = TargetMapName;
			PendingTravelChunkId = ResolvedChunkId;
			bPendingTravelUsesServerTravel = bServerTravel;
			EnsureAwsChunksReady({ResolvedChunkId});
			if (bPromptModalIfUnmounted)
			{
				ShowOnDemandDownloadModal(ResolvedChunkId);
			}
			return false;
		}
	}

	// Depot rule for the internal CDN. Chunks 0 and 1001 open here; other chunks open only when mounted.
	const bool bIsCoreOrAlreadyMounted = (ResolvedChunkId <= 0 || ResolvedChunkId == 1001 || IsChunkMounted(ResolvedChunkId));

	if (bIsCoreOrAlreadyMounted)
	{
		UE_LOG(LogChunkManager, Log, TEXT("RequestLevelTravelWithChunkGuard: Chunk %d for '%s' is mounted. Executing travel."),
			ResolvedChunkId, *TargetMapName.ToString());
		TravelToResolvedMap(TargetMapName, bServerTravel);
		return true;
	}

	// Chunk is NOT mounted: Intercept travel and initiate graceful flow
	UE_LOG(LogChunkManager, Log, TEXT("RequestLevelTravelWithChunkGuard: Chunk %d for '%s' is UNMOUNTED. Intercepting travel."),
		ResolvedChunkId, *TargetMapName.ToString());

	// Verify Storage Headroom (Guard 3)
	int64 RequiredBytes = 0;
	int64 FreeBytes = 0;
	if (!CheckStorageHeadroom(ResolvedChunkId, RequiredBytes, FreeBytes))
	{
		const FText ErrorMsg = FText::Format(
			NSLOCTEXT("ChunkManager", "StorageFull", "Bộ nhớ thiết bị không đủ dung lượng. Cần thêm dung lượng để tải bản đồ."),
			FText::AsMemory(RequiredBytes), FText::AsMemory(FreeBytes));
		OnChunkDownloadError.Broadcast(ResolvedChunkId, ErrorMsg);
		return false;
	}

	// Cache pending travel target
	PendingTravelMapName = TargetMapName;
	PendingTravelChunkId = ResolvedChunkId;
	bPendingTravelUsesServerTravel = bServerTravel;

	// Enqueue parent chunk if needed (Guard 2: Cascade Dependency)
	const int32 ParentId = GetParentChunkId(ResolvedChunkId);
	if (ParentId > 0 && !IsChunkMounted(ParentId))
	{
		UE_LOG(LogChunkManager, Log, TEXT("RequestLevelTravelWithChunkGuard: Auto-enqueuing parent chunk %d before child %d."),
			ParentId, ResolvedChunkId);
		EnqueueChunkDownload(ParentId, EChunkManagerPriority::P0_Foreground);
	}

	// Promote target chunk to P0 Foreground (Guard 4: Foreground Preemption)
	EnqueueChunkDownload(ResolvedChunkId, EChunkManagerPriority::P0_Foreground);

	// Display On-Demand Modal if requested
	if (bPromptModalIfUnmounted)
	{
		ShowOnDemandDownloadModal(ResolvedChunkId);
	}

	return false;
}
```

Một pak vừa mount. URL S3 đợi cả cụm chunk đang chờ rồi mới travel.

`PrototypeRacing/Plugins/ChunkDownloaderCustom/Source/ChunkDownloaderCustom/Private/ChunkDownloadManagerSubsystem.cpp`

```cpp
void UChunkDownloadManagerSubsystem::HandleResourcePackMounted(int32 ChunkId)
{
	UE_LOG(LogChunkManager, Log, TEXT("HandleResourcePackMounted: Chunk %d mount completed."), ChunkId);

	Queue_P0_Foreground.Remove(ChunkId);
	Queue_P1_PostRace.Remove(ChunkId);
	Queue_P2_Background.Remove(ChunkId);

	if (ActiveForegroundChunkId == ChunkId)
	{
		ActiveForegroundChunkId = INDEX_NONE;
	}

	OnChunkStatusChanged.Broadcast(ChunkId, EChunkManagerStatus::Mounted);
	OnChunkMountCompleted.Broadcast(ChunkId, true);

	// Seamless Travel Resolution
	const UChunkDownloaderSubsystem* Transport = GetTransportSubsystem();
	const bool bAws = Transport && Transport->IsAwsCdnEnabled();
	const bool bPendingReady = bAws ? AreAwsChunksReady({PendingTravelChunkId}) : PendingTravelChunkId == ChunkId;
	if (!PendingTravelMapName.IsNone() && bPendingReady)
	{
		const FName TargetMap = PendingTravelMapName;
		const bool bServerTravel = bPendingTravelUsesServerTravel;
		PendingTravelMapName = NAME_None;
		PendingTravelChunkId = INDEX_NONE;
		bPendingTravelUsesServerTravel = false;

		UE_LOG(LogChunkManager, Log, TEXT("HandleResourcePackMounted: Required chunk %d mounted. Executing deferred travel to '%s'."),
			ChunkId, *TargetMap.ToString());
		TravelToResolvedMap(TargetMap, bServerTravel);
	}

	ProcessNextQueueItem();
}
```

Waiting room trên S3 đi qua guard. URL khác vẫn `ServerTravel`.

`PrototypeRacing/Source/PrototypeRacing/Private/Multiplayer/MultiplayerWaitingRoomGameMode.cpp`

```cpp
	if (UGameInstance* GameInstance = GetGameInstance())
	{
		if (UChunkDownloaderSubsystem* Transport = GameInstance->GetSubsystem<UChunkDownloaderSubsystem>();
			Transport && Transport->IsAwsCdnEnabled())
		{
			if (UChunkDownloadManagerSubsystem* ChunkManager = GameInstance->GetSubsystem<UChunkDownloadManagerSubsystem>())
			{
				ChunkManager->RequestLevelTravelWithChunkGuard(FName(*TargetMap), -1, false, true);
				return;
			}
		}
	}

	World->ServerTravel(TargetMap);
```

