# VoxelMap 缓存共享系统实现计划

## 项目概述

本项目旨在实现一个基于Velocity代理服务器的VoxelMap缓存共享系统，允许玩家之间共享地图数据。系统通过直接操作VoxelMap缓存文件实现，无需修改原始mod代码。

## 项目结构

```
VoxelMapCacheShare/
├── velocity-plugin/
│   ├── src/main/java/
│   │   └── com/example/voxelcache/
│   │       ├── VoxelMapCachePlugin.java
│   │       ├── CacheManager.java
│   │       ├── ConflictResolver.java
│   │       ├── NetworkHandler.java
│   │       └── models/
│   │           ├── CacheData.java
│   │           ├── CacheMetadata.java
│   │           └── RegionInfo.java
│   ├── pom.xml
│   └── plugin.yml
├── client-patch/
│   ├── src/main/java/
│   │   └── com/example/voxelclient/
│   │       ├── VoxelMapClient.java
│   │       ├── FileWatcher.java
│   │       ├── NetworkClient.java
│   │       ├── CacheProcessor.java
│   │       └── config/
│   │           └── ClientConfig.java
│   ├── pom.xml
│   └── fabric.mod.json
└── shared/
    └── src/main/java/
        └── com/example/voxelshared/
            ├── PacketTypes.java
            ├── CompressionUtils.java
            └── CacheUtils.java
```

## 实现阶段

### 阶段一：基础架构设计

#### 1.1 数据模型定义

```java
// shared/CacheData.java
public class CacheData {
    private final String worldName;
    private final String subworldName;
    private final String dimensionName;
    private final String regionKey;
    private final long timestamp;
    private final UUID uploaderId;
    private final byte[] compressedData;
    private final CacheMetadata metadata;
    
    public CacheData(String worldName, String subworldName, String dimensionName, 
                    String regionKey, long timestamp, UUID uploaderId, 
                    byte[] compressedData, CacheMetadata metadata) {
        this.worldName = worldName;
        this.subworldName = subworldName;
        this.dimensionName = dimensionName;
        this.regionKey = regionKey;
        this.timestamp = timestamp;
        this.uploaderId = uploaderId;
        this.compressedData = compressedData;
        this.metadata = metadata;
    }
    
    // Getters...
}

// shared/CacheMetadata.java
public class CacheMetadata {
    private final int version;
    private final long creationTime;
    private final Map<String, Long> chunkTimestamps;
    private final int dataScore;
    private final String serverIdentifier;
    
    public CacheMetadata(int version, long creationTime, 
                        Map<String, Long> chunkTimestamps, 
                        int dataScore, String serverIdentifier) {
        this.version = version;
        this.creationTime = creationTime;
        this.chunkTimestamps = new HashMap<>(chunkTimestamps);
        this.dataScore = dataScore;
        this.serverIdentifier = serverIdentifier;
    }
    
    public boolean isNewerThan(CacheMetadata other) {
        if (this.version != other.version) {
            return this.version > other.version;
        }
        return this.creationTime > other.creationTime;
    }
    
    public boolean isMoreCompleteThan(CacheMetadata other) {
        return this.dataScore > other.dataScore * 1.1; // 10% threshold
    }
}
```

#### 1.2 网络协议定义

```java
// shared/PacketTypes.java
public enum PacketType {
    CACHE_UPLOAD_REQUEST(0x01),
    CACHE_UPLOAD_RESPONSE(0x02),
    CACHE_DOWNLOAD_REQUEST(0x03),
    CACHE_DOWNLOAD_RESPONSE(0x04),
    CACHE_LIST_REQUEST(0x05),
    CACHE_LIST_RESPONSE(0x06),
    CACHE_NOTIFICATION(0x07),
    CACHE_CONFLICT_RESOLUTION(0x08);
    
    private final int id;
    
    PacketType(int id) {
        this.id = id;
    }
    
    public int getId() {
        return id;
    }
    
    public static PacketType fromId(int id) {
        for (PacketType type : values()) {
            if (type.id == id) {
                return type;
            }
        }
        throw new IllegalArgumentException("Unknown packet type: " + id);
    }
}
```

### 阶段二：Velocity插件开发

#### 2.1 主插件类

```java
// velocity-plugin/VoxelMapCachePlugin.java
@Plugin(
    id = "voxelmap-cache-share",
    name = "VoxelMap Cache Share",
    version = "1.0.0",
    description = "Share VoxelMap cache data between players",
    authors = {"YourName"}
)
public class VoxelMapCachePlugin {
    private static final ChannelIdentifier CACHE_CHANNEL = 
        MinecraftChannelIdentifier.from("voxelcache:main");
    
    private final ProxyServer server;
    private final Logger logger;
    private final Path dataDirectory;
    private CacheManager cacheManager;
    private NetworkHandler networkHandler;
    private ConflictResolver conflictResolver;
    
    @Inject
    public VoxelMapCachePlugin(ProxyServer server, Logger logger, 
                              @DataDirectory Path dataDirectory) {
        this.server = server;
        this.logger = logger;
        this.dataDirectory = dataDirectory;
    }
    
    @Subscribe
    public void onProxyInitialization(ProxyInitializeEvent event) {
        // 初始化组件
        this.cacheManager = new CacheManager(dataDirectory, logger);
        this.conflictResolver = new ConflictResolver(logger);
        this.networkHandler = new NetworkHandler(server, cacheManager, 
                                                conflictResolver, logger);
        
        // 注册插件消息通道
        server.getChannelRegistrar().register(CACHE_CHANNEL);
        
        // 注册事件监听器
        server.getEventManager().register(this, networkHandler);
        
        logger.info("VoxelMap Cache Share plugin initialized");
    }
    
    @Subscribe
    public void onPluginMessage(PluginMessageEvent event) {
        if (!event.getIdentifier().equals(CACHE_CHANNEL)) {
            return;
        }
        
        if (!(event.getSource() instanceof Player)) {
            return;
        }
        
        Player player = (Player) event.getSource();
        networkHandler.handleMessage(player, event.getData());
        event.setResult(PluginMessageEvent.ForwardResult.handled());
    }
    
    @Subscribe
    public void onPlayerDisconnect(DisconnectEvent event) {
        // 清理玩家相关的缓存请求
        networkHandler.cleanupPlayer(event.getPlayer().getUniqueId());
    }
}
```

#### 2.2 缓存管理器

```java
// velocity-plugin/CacheManager.java
public class CacheManager {
    private final Path dataDirectory;
    private final Logger logger;
    private final Map<String, CacheMetadata> cacheIndex;
    private final ExecutorService executorService;
    private final ReadWriteLock indexLock;
    
    public CacheManager(Path dataDirectory, Logger logger) {
        this.dataDirectory = dataDirectory;
        this.logger = logger;
        this.cacheIndex = new ConcurrentHashMap<>();
        this.executorService = Executors.newFixedThreadPool(4);
        this.indexLock = new ReentrantReadWriteLock();
        
        loadCacheIndex();
        startMaintenanceTask();
    }
    
    public CompletableFuture<Boolean> saveCacheData(CacheData cacheData) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                String cacheKey = buildCacheKey(cacheData);
                
                // 检查是否需要更新
                CacheMetadata existing = getCacheMetadata(cacheKey);
                if (existing != null && !shouldUpdate(existing, cacheData.getMetadata())) {
                    logger.debug("Cache data not updated - existing data is newer: {}", cacheKey);
                    return false;
                }
                
                // 创建目录结构
                Path cachePath = buildCachePath(cacheData);
                Files.createDirectories(cachePath.getParent());
                
                // 保存数据文件
                Files.write(cachePath, cacheData.getCompressedData(), 
                           StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING);
                
                // 更新索引
                indexLock.writeLock().lock();
                try {
                    cacheIndex.put(cacheKey, cacheData.getMetadata());
                    saveCacheIndex();
                } finally {
                    indexLock.writeLock().unlock();
                }
                
                logger.info("Cache data saved: {} (uploaded by {})", 
                           cacheKey, cacheData.getUploaderId());
                return true;
                
            } catch (Exception e) {
                logger.error("Failed to save cache data", e);
                return false;
            }
        }, executorService);
    }
    
    public CompletableFuture<Optional<CacheData>> getCacheData(String worldName, 
                                                              String subworldName,
                                                              String dimensionName, 
                                                              String regionKey) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                String cacheKey = buildCacheKey(worldName, subworldName, dimensionName, regionKey);
                
                indexLock.readLock().lock();
                CacheMetadata metadata;
                try {
                    metadata = cacheIndex.get(cacheKey);
                } finally {
                    indexLock.readLock().unlock();
                }
                
                if (metadata == null) {
                    return Optional.empty();
                }
                
                Path cachePath = buildCachePath(worldName, subworldName, dimensionName, regionKey);
                if (!Files.exists(cachePath)) {
                    logger.warn("Cache metadata exists but file missing: {}", cacheKey);
                    return Optional.empty();
                }
                
                byte[] data = Files.readAllBytes(cachePath);
                CacheData cacheData = new CacheData(worldName, subworldName, dimensionName,
                                                   regionKey, metadata.getCreationTime(),
                                                   UUID.randomUUID(), data, metadata);
                
                return Optional.of(cacheData);
                
            } catch (Exception e) {
                logger.error("Failed to read cache data", e);
                return Optional.empty();
            }
        }, executorService);
    }
    
    public List<RegionInfo> getAvailableRegions(String worldName, String subworldName, 
                                               String dimensionName) {
        List<RegionInfo> regions = new ArrayList<>();
        String prefix = buildCacheKeyPrefix(worldName, subworldName, dimensionName);
        
        indexLock.readLock().lock();
        try {
            for (Map.Entry<String, CacheMetadata> entry : cacheIndex.entrySet()) {
                if (entry.getKey().startsWith(prefix)) {
                    String regionKey = extractRegionKey(entry.getKey());
                    regions.add(new RegionInfo(regionKey, entry.getValue()));
                }
            }
        } finally {
            indexLock.readLock().unlock();
        }
        
        return regions;
    }
    
    private boolean shouldUpdate(CacheMetadata existing, CacheMetadata incoming) {
        // 版本号检查
        if (incoming.getVersion() > existing.getVersion()) {
            return true;
        }
        
        // 时间戳检查
        if (incoming.isNewerThan(existing)) {
            return true;
        }
        
        // 完整性检查
        if (incoming.isMoreCompleteThan(existing)) {
            return true;
        }
        
        return false;
    }
    
    private void startMaintenanceTask() {
        // 定期清理过期缓存和维护索引
        ScheduledExecutorService scheduler = Executors.newSingleThreadScheduledExecutor();
        scheduler.scheduleAtFixedRate(this::performMaintenance, 1, 1, TimeUnit.HOURS);
    }
    
    private void performMaintenance() {
        try {
            // 清理过期缓存 (超过30天)
            long cutoffTime = System.currentTimeMillis() - TimeUnit.DAYS.toMillis(30);
            
            indexLock.writeLock().lock();
            try {
                Iterator<Map.Entry<String, CacheMetadata>> iterator = cacheIndex.entrySet().iterator();
                while (iterator.hasNext()) {
                    Map.Entry<String, CacheMetadata> entry = iterator.next();
                    if (entry.getValue().getCreationTime() < cutoffTime) {
                        // 删除文件
                        String cacheKey = entry.getKey();
                        Path cachePath = buildCachePathFromKey(cacheKey);
                        try {
                            Files.deleteIfExists(cachePath);
                            iterator.remove();
                            logger.debug("Cleaned up expired cache: {}", cacheKey);
                        } catch (IOException e) {
                            logger.warn("Failed to delete expired cache file: {}", cachePath, e);
                        }
                    }
                }
                saveCacheIndex();
            } finally {
                indexLock.writeLock().unlock();
            }
            
            logger.info("Cache maintenance completed");
        } catch (Exception e) {
            logger.error("Cache maintenance failed", e);
        }
    }
    
    // 辅助方法...
    private String buildCacheKey(CacheData cacheData) {
        return buildCacheKey(cacheData.getWorldName(), cacheData.getSubworldName(),
                           cacheData.getDimensionName(), cacheData.getRegionKey());
    }
    
    private String buildCacheKey(String worldName, String subworldName, 
                               String dimensionName, String regionKey) {
        return String.format("%s/%s/%s/%s", 
                           sanitizeFileName(worldName),
                           sanitizeFileName(subworldName), 
                           sanitizeFileName(dimensionName),
                           regionKey);
    }
    
    private Path buildCachePath(CacheData cacheData) {
        return buildCachePath(cacheData.getWorldName(), cacheData.getSubworldName(),
                            cacheData.getDimensionName(), cacheData.getRegionKey());
    }
    
    private Path buildCachePath(String worldName, String subworldName, 
                              String dimensionName, String regionKey) {
        return dataDirectory
            .resolve("cache")
            .resolve(sanitizeFileName(worldName))
            .resolve(sanitizeFileName(subworldName))
            .resolve(sanitizeFileName(dimensionName))
            .resolve(regionKey + ".zip");
    }
    
    private String sanitizeFileName(String name) {
        if (name == null || name.isEmpty()) {
            return "default";
        }
        return name.replaceAll("[^a-zA-Z0-9._-]", "_");
    }
}
```

#### 2.3 冲突解决器

```java
// velocity-plugin/ConflictResolver.java
public class ConflictResolver {
    private final Logger logger;
    
    public ConflictResolver(Logger logger) {
        this.logger = logger;
    }
    
    public CompletableFuture<CacheData> resolveConflict(CacheData existing, CacheData incoming) {
        return CompletableFuture.supplyAsync(() -> {
            try {
                logger.info("Resolving cache conflict between {} and {}", 
                           existing.getUploaderId(), incoming.getUploaderId());
                
                // 1. 简单时间戳比较
                long timeDiff = Math.abs(existing.getTimestamp() - incoming.getTimestamp());
                if (timeDiff > TimeUnit.MINUTES.toMillis(5)) {
                    CacheData newer = existing.getTimestamp() > incoming.getTimestamp() ? 
                                    existing : incoming;
                    logger.info("Conflict resolved by timestamp - using data from {}", 
                               newer.getUploaderId());
                    return newer;
                }
                
                // 2. 版本比较
                if (existing.getMetadata().getVersion() != incoming.getMetadata().getVersion()) {
                    CacheData newerVersion = existing.getMetadata().getVersion() > 
                                           incoming.getMetadata().getVersion() ? existing : incoming;
                    logger.info("Conflict resolved by version - using version {} from {}", 
                               newerVersion.getMetadata().getVersion(), 
                               newerVersion.getUploaderId());
                    return newerVersion;
                }
                
                // 3. 数据完整性比较
                if (existing.getMetadata().isMoreCompleteThan(incoming.getMetadata())) {
                    logger.info("Conflict resolved by completeness - keeping existing data");
                    return existing;
                } else if (incoming.getMetadata().isMoreCompleteThan(existing.getMetadata())) {
                    logger.info("Conflict resolved by completeness - using incoming data");
                    return incoming;
                }
                
                // 4. 智能合并
                return performIntelligentMerge(existing, incoming);
                
            } catch (Exception e) {
                logger.error("Conflict resolution failed, keeping existing data", e);
                return existing;
            }
        });
    }
    
    private CacheData performIntelligentMerge(CacheData data1, CacheData data2) {
        try {
            logger.info("Performing intelligent merge of cache data");
            
            // 解析ZIP文件内容
            Map<String, byte[]> content1 = extractZipContent(data1.getCompressedData());
            Map<String, byte[]> content2 = extractZipContent(data2.getCompressedData());
            
            // 合并数据
            Map<String, byte[]> mergedContent = new HashMap<>();
            
            // 合并control文件（版本信息）
            mergedContent.put("control", mergeControlData(content1.get("control"), 
                                                         content2.get("control")));
            
            // 合并key文件（方块状态映射）
            mergedContent.put("key", mergeKeyData(content1.get("key"), 
                                                 content2.get("key")));
            
            // 合并data文件（实际地图数据）
            mergedContent.put("data", mergeMapData(content1.get("data"), 
                                                  content2.get("data"),
                                                  content1.get("key"),
                                                  content2.get("key")));
            
            // 重新打包为ZIP
            byte[] mergedZip = createZipFromContent(mergedContent);
            
            // 创建合并后的元数据
            CacheMetadata mergedMetadata = createMergedMetadata(data1.getMetadata(), 
                                                              data2.getMetadata());
            
            CacheData merged = new CacheData(
                data1.getWorldName(),
                data1.getSubworldName(), 
                data1.getDimensionName(),
                data1.getRegionKey(),
                Math.max(data1.getTimestamp(), data2.getTimestamp()),
                data1.getUploaderId(), // 保持原始上传者
                mergedZip,
                mergedMetadata
            );
            
            logger.info("Intelligent merge completed successfully");
            return merged;
            
        } catch (Exception e) {
            logger.error("Intelligent merge failed, falling back to newer data", e);
            return data1.getTimestamp() > data2.getTimestamp() ? data1 : data2;
        }
    }
    
    private Map<String, byte[]> extractZipContent(byte[] zipData) throws IOException {
        Map<String, byte[]> content = new HashMap<>();
        
        try (ByteArrayInputStream bis = new ByteArrayInputStream(zipData);
             ZipInputStream zis = new ZipInputStream(bis)) {
            
            ZipEntry entry;
            while ((entry = zis.getNextEntry()) != null) {
                ByteArrayOutputStream baos = new ByteArrayOutputStream();
                byte[] buffer = new byte[1024];
                int bytesRead;
                
                while ((bytesRead = zis.read(buffer)) != -1) {
                    baos.write(buffer, 0, bytesRead);
                }
                
                content.put(entry.getName(), baos.toByteArray());
                zis.closeEntry();
            }
        }
        
        return content;
    }
    
    private byte[] createZipFromContent(Map<String, byte[]> content) throws IOException {
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        
        try (ZipOutputStream zos = new ZipOutputStream(baos)) {
            for (Map.Entry<String, byte[]> entry : content.entrySet()) {
                ZipEntry zipEntry = new ZipEntry(entry.getKey());
                zipEntry.setSize(entry.getValue().length);
                zos.putNextEntry(zipEntry);
                zos.write(entry.getValue());
                zos.closeEntry();
            }
        }
        
        return baos.toByteArray();
    }
    
    // 其他合并方法的实现...
    private byte[] mergeControlData(byte[] control1, byte[] control2) {
        // 简单选择版本号较高的control数据
        try {
            Properties props1 = new Properties();
            Properties props2 = new Properties();
            
            if (control1 != null) {
                props1.load(new ByteArrayInputStream(control1));
            }
            if (control2 != null) {
                props2.load(new ByteArrayInputStream(control2));
            }
            
            int version1 = Integer.parseInt(props1.getProperty("version", "1"));
            int version2 = Integer.parseInt(props2.getProperty("version", "1"));
            
            return version1 >= version2 ? control1 : control2;
            
        } catch (Exception e) {
            logger.warn("Failed to parse control data, using first one", e);
            return control1 != null ? control1 : control2;
        }
    }
    
    private byte[] mergeKeyData(byte[] key1, byte[] key2) {
        // 合并方块状态映射表
        try {
            Set<String> allLines = new HashSet<>();
            
            if (key1 != null) {
                String[] lines1 = new String(key1, StandardCharsets.UTF_8).split("\n");
                Collections.addAll(allLines, lines1);
            }
            
            if (key2 != null) {
                String[] lines2 = new String(key2, StandardCharsets.UTF_8).split("\n");
                Collections.addAll(allLines, lines2);
            }
            
            return String.join("\n", allLines).getBytes(StandardCharsets.UTF_8);
            
        } catch (Exception e) {
            logger.warn("Failed to merge key data, using first one", e);
            return key1 != null ? key1 : key2;
        }
    }
    
    private byte[] mergeMapData(byte[] data1, byte[] data2, byte[] key1, byte[] key2) {
        // 这里需要实现复杂的地图数据合并逻辑
        // 暂时返回数据量较大的那个
        if (data1 == null) return data2;
        if (data2 == null) return data1;
        
        return data1.length >= data2.length ? data1 : data2;
    }
    
    private CacheMetadata createMergedMetadata(CacheMetadata meta1, CacheMetadata meta2) {
        int maxVersion = Math.max(meta1.getVersion(), meta2.getVersion());
        long latestTime = Math.max(meta1.getCreationTime(), meta2.getCreationTime());
        int maxScore = Math.max(meta1.getDataScore(), meta2.getDataScore());
        
        // 合并区块时间戳
        Map<String, Long> mergedTimestamps = new HashMap<>(meta1.getChunkTimestamps());
        for (Map.Entry<String, Long> entry : meta2.getChunkTimestamps().entrySet()) {
            mergedTimestamps.merge(entry.getKey(), entry.getValue(), Long::max);
        }
        
        return new CacheMetadata(maxVersion, latestTime, mergedTimestamps, 
                               maxScore, meta1.getServerIdentifier());
    }
}
```

### 阶段三：客户端补丁开发

#### 3.1 文件监控器

```java
// client-patch/FileWatcher.java
public class FileWatcher {
    private static final Logger LOGGER = LoggerFactory.getLogger(FileWatcher.class);
    private final NetworkClient networkClient;
    private final CacheProcessor cacheProcessor;
    private final Path voxelMapCacheDir;
    private final WatchService watchService;
    private final Map<WatchKey, Path> watchKeys;
    private final ExecutorService executorService;
    private volatile boolean running;
    
    public FileWatcher(NetworkClient networkClient, CacheProcessor cacheProcessor) {
        this.networkClient = networkClient;
        this.cacheProcessor = cacheProcessor;
        this.voxelMapCacheDir = getVoxelMapCacheDirectory();
        this.watchKeys = new ConcurrentHashMap<>();
        this.executorService = Executors.newSingleThreadExecutor(r -> {
            Thread thread = new Thread(r, "VoxelCache-FileWatcher");
            thread.setDaemon(true);
            return thread;
        });
        
        try {
            this.watchService = FileSystems.getDefault().newWatchService();
            setupWatchers();
        } catch (IOException e) {
            throw new RuntimeException("Failed to initialize file watcher", e);
        }
    }
    
    public void start() {
        if (running) {
            return;
        }
        
        running = true;
        executorService.submit(this::watchForChanges);
        LOGGER.info("File watcher started, monitoring: {}", voxelMapCacheDir);
    }
    
    public void stop() {
        running = false;
        
        try {
            watchService.close();
            executorService.shutdown();
            if (!executorService.awaitTermination(5, TimeUnit.SECONDS)) {
                executorService.shutdownNow();
            }
        } catch (Exception e) {
            LOGGER.error("Error stopping file watcher", e);
        }
        
        LOGGER.info("File watcher stopped");
    }
    
    private void setupWatchers() throws IOException {
        if (!Files.exists(voxelMapCacheDir)) {
            Files.createDirectories(voxelMapCacheDir);
        }
        
        // 递归监控所有子目录
        Files.walkFileTree(voxelMapCacheDir, new SimpleFileVisitor<Path>() {
            @Override
            public FileVisitResult preVisitDirectory(Path dir, BasicFileAttributes attrs) 
                    throws IOException {
                registerDirectory(dir);
                return FileVisitResult.CONTINUE;
            }
        });
    }
    
    private void registerDirectory(Path dir) throws IOException {
        LOGGER.debug("Registering directory for watching: {}", dir);
        WatchKey key = dir.register(watchService,
            StandardWatchEventKinds.ENTRY_CREATE,
            StandardWatchEventKinds.ENTRY_MODIFY,
            StandardWatchEventKinds.ENTRY_DELETE);
        watchKeys.put(key, dir);
    }
    
    private void watchForChanges() {
        while (running) {
            try {
                WatchKey key = watchService.poll(1, TimeUnit.SECONDS);
                if (key == null) {
                    continue;
                }
                
                Path dir = watchKeys.get(key);
                if (dir == null) {
                    continue;
                }
                
                for (WatchEvent<?> event : key.pollEvents()) {
                    WatchEvent.Kind<?> kind = event.kind();
                    
                    if (kind == StandardWatchEventKinds.OVERFLOW) {
                        continue;
                    }
                    
                    @SuppressWarnings("unchecked")
                    WatchEvent<Path> ev = (WatchEvent<Path>) event;
                    Path filename = ev.context();
                    Path fullPath = dir.resolve(filename);
                    
                    handleFileEvent(kind, fullPath);
                }
                
                boolean valid = key.reset();
                if (!valid) {
                    watchKeys.remove(key);
                    LOGGER.warn("Watch key no longer valid for directory: {}", dir);
                }
                
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            } catch (Exception e) {
                LOGGER.error("Error in file watcher", e);
            }
        }
    }
    
    private void handleFileEvent(WatchEvent.Kind<?> kind, Path filePath) {
        try {
            String fileName = filePath.getFileName().toString();
            
            // 只处理.zip文件
            if (!fileName.endsWith(".zip")) {
                return;
            }
            
            // 新建目录时注册监控
            if (kind == StandardWatchEventKinds.ENTRY_CREATE && Files.isDirectory(filePath)) {
                registerDirectory(filePath);
                return;
            }
            
            // 处理缓存文件变化
            if (kind == StandardWatchEventKinds.ENTRY_CREATE || 
                kind == StandardWatchEventKinds.ENTRY_MODIFY) {
                
                // 延迟处理，避免文件正在写入时读取
                executorService.schedule(() -> {
                    if (Files.exists(filePath)) {
                        handleCacheFileUpdate(filePath);
                    }
                }, 2, TimeUnit.SECONDS);
                
            } else if (kind == StandardWatchEventKinds.ENTRY_DELETE) {
                handleCacheFileDelete(filePath);
            }
            
        } catch (Exception e) {
            LOGGER.error("Error handling file event for: {}", filePath, e);
        }
    }
    
    private void handleCacheFileUpdate(Path filePath) {
        try {
            LOGGER.debug("Processing cache file update: {}", filePath);
            
            // 解析文件路径获取世界信息
            RegionInfo regionInfo = parseRegionInfo(filePath);
            if (regionInfo == null) {
                return;
            }
            
            // 读取文件内容
            byte[] fileData = Files.readAllBytes(filePath);
            
            // 处理缓存数据
            cacheProcessor.processCacheFile(regionInfo, fileData, CacheOperation.UPLOAD);
            
        } catch (Exception e) {
            LOGGER.error("Failed to handle cache file update: {}", filePath, e);
        }
    }
    
    private void handleCacheFileDelete(Path filePath) {
        try {
            LOGGER.debug("Processing cache file delete: {}", filePath);
            
            RegionInfo regionInfo = parseRegionInfo(filePath);
            if (regionInfo == null) {
                return;
            }
            
            cacheProcessor.processCacheFile(regionInfo, null, CacheOperation.DELETE);
            
        } catch (Exception e) {
            LOGGER.error("Failed to handle cache file delete: {}", filePath, e);
        }
    }
    
    private RegionInfo parseRegionInfo(Path filePath) {
        try {
            Path relativePath = voxelMapCacheDir.relativize(filePath);
            String[] parts = relativePath.toString().split(Pattern.quote(File.separator));
            
            if (parts.length < 4) {
                LOGGER.warn("Invalid cache file path structure: {}", filePath);
                return null;
            }
            
            String worldName = parts[0];
            String subworldName = parts[1];
            String dimensionName = parts[2];
            String fileName = parts[3];
            
            // 提取区域坐标
            String regionKey = fileName.substring(0, fileName.lastIndexOf('.'));
            
            return new RegionInfo(worldName, subworldName, dimensionName, regionKey);
            
        } catch (Exception e) {
            LOGGER.error("Failed to parse region info from path: {}", filePath, e);
            return null;
        }
    }
    
    private Path getVoxelMapCacheDirectory() {
        // 获取Minecraft客户端目录下的voxelmap/cache目录
        Path minecraftDir = Paths.get(System.getProperty("user.dir"));
        return minecraftDir.resolve("voxelmap").resolve("cache");
    }
    
    // 辅助类
    public static class RegionInfo {
        private final String worldName;
        private final String subworldName;
        private final String dimensionName;
        private final String regionKey;
        
        public RegionInfo(String worldName, String subworldName, 
                         String dimensionName, String regionKey) {
            this.worldName = worldName;
            this.subworldName = subworldName;
            this.dimensionName = dimensionName;
            this.regionKey = regionKey;
        }
        
        // Getters...
        public String getWorldName() { return worldName; }
        public String getSubworldName() { return subworldName; }
        public String getDimensionName() { return dimensionName; }
        public String getRegionKey() { return regionKey; }
    }
    
    public enum CacheOperation {
        UPLOAD, DOWNLOAD, DELETE
    }
}
```

#### 3.2 网络客户端

```java
// client-patch/NetworkClient.java
public class NetworkClient {
    private static final Logger LOGGER = LoggerFactory.getLogger(NetworkClient.class);
    private static final String CHANNEL_NAME = "voxelcache:main";
    
    private final ClientConfig config;
    private final Map<String, CompletableFuture<CacheData>> pendingRequests;
    private final ScheduledExecutorService scheduler;
    private final ExecutorService networkExecutor;
    private volatile boolean connected;
    
    public NetworkClient(ClientConfig config) {
        this.config = config;
        this.pendingRequests = new ConcurrentHashMap<>();
        this.scheduler = Executors.newScheduledThreadPool(2);
        this.networkExecutor = Executors.newFixedThreadPool(4);
        this.connected = false;
        
        // 注册网络通道
        registerNetworkChannel();
    }
    
    private void registerNetworkChannel() {
        // 这里需要根据具体的mod loader实现网络注册
        // 示例使用Fabric的网络API
        /*
        ClientPlayNetworking.registerGlobalReceiver(
            new Identifier("voxelcache", "main"), 
            this::handleIncomingPacket
        );
        */
    }
    
    public CompletableFuture<Boolean> uploadCacheData(CacheData cacheData) {
        if (!connected || !config.isAutoUploadEnabled()) {
            return CompletableFuture.completedFuture(false);
        }
        
        return CompletableFuture.supplyAsync(() -> {
            try {
                LOGGER.debug("Uploading cache data: {}/{}/{}/{}", 
                           cacheData.getWorldName(), cacheData.getSubworldName(),
                           cacheData.getDimensionName(), cacheData.getRegionKey());
                
                ByteArrayOutputStream baos = new ByteArrayOutputStream();
                DataOutputStream dos = new DataOutputStream(baos);
                
                // 写入数据包类型
                dos.writeByte(PacketType.CACHE_UPLOAD_REQUEST.getId());
                
                // 写入数据
                dos.writeUTF(cacheData.getWorldName());
                dos.writeUTF(cacheData.getSubworldName());
                dos.writeUTF(cacheData.getDimensionName());
                dos.writeUTF(cacheData.getRegionKey());
                dos.writeLong(cacheData.getTimestamp());
                
                // 写入元数据
                writeMetadata(dos, cacheData.getMetadata());
                
                // 写入压缩数据
                dos.writeInt(cacheData.getCompressedData().length);
                dos.write(cacheData.getCompressedData());
                
                // 发送数据包
                sendPacket(baos.toByteArray());
                
                return true;
                
            } catch (Exception e) {
                LOGGER.error("Failed to upload cache data", e);
                return false;
            }
        }, networkExecutor);
    }
    
    public CompletableFuture<Optional<CacheData>> requestCacheData(String worldName,
                                                                  String subworldName,
                                                                  String dimensionName,
                                                                  String regionKey) {
        if (!connected || !config.isAutoDownloadEnabled()) {
            return CompletableFuture.completedFuture(Optional.empty());
        }
        
        String requestKey = buildRequestKey(worldName, subworldName, dimensionName, regionKey);
        
        // 检查是否已有请求在进行中
        CompletableFuture<CacheData> existing = pendingRequests.get(requestKey);
        if (existing != null) {
            return existing.thenApply(Optional::ofNullable);
        }
        
        CompletableFuture<CacheData> future = new CompletableFuture<>();
        pendingRequests.put(requestKey, future);
        
        // 设置超时
        scheduler.schedule(() -> {
            if (!future.isDone()) {
                future.completeExceptionally(new TimeoutException("Cache request timeout"));
                pendingRequests.remove(requestKey);
            }
        }, config.getRequestTimeout(), TimeUnit.MILLISECONDS);
        
        // 发送请求
        networkExecutor.execute(() -> {
            try {
                ByteArrayOutputStream baos = new ByteArrayOutputStream();
                DataOutputStream dos = new DataOutputStream(baos);
                
                dos.writeByte(PacketType.CACHE_DOWNLOAD_REQUEST.getId());
                dos.writeUTF(worldName);
                dos.writeUTF(subworldName);
                dos.writeUTF(dimensionName);
                dos.writeUTF(regionKey);
                
                sendPacket(baos.toByteArray());
                
            } catch (Exception e) {
                LOGGER.error("Failed to request cache data", e);
                future.completeExceptionally(e);
                pendingRequests.remove(requestKey);
            }
        });
        
        return future.thenApply(Optional::ofNullable);
    }
    
    public CompletableFuture<List<RegionInfo>> requestAvailableRegions(String worldName,
                                                                      String subworldName,
                                                                      String dimensionName) {
        if (!connected) {
            return CompletableFuture.completedFuture(Collections.emptyList());
        }
        
        return CompletableFuture.supplyAsync(() -> {
            try {
                ByteArrayOutputStream baos = new ByteArrayOutputStream();
                DataOutputStream dos = new DataOutputStream(baos);
                
                dos.writeByte(PacketType.CACHE_LIST_REQUEST.getId());
                dos.writeUTF(worldName);
                dos.writeUTF(subworldName);
                dos.writeUTF(dimensionName);
                
                sendPacket(baos.toByteArray());
                
                // 这里简化处理，实际应该等待响应
                return Collections.emptyList();
                
            } catch (Exception e) {
                LOGGER.error("Failed to request available regions", e);
                return Collections.emptyList();
            }
        }, networkExecutor);
    }
    
    private void handleIncomingPacket(byte[] data) {
        try {
            ByteArrayInputStream bais = new ByteArrayInputStream(data);
            DataInputStream dis = new DataInputStream(bais);
            
            PacketType type = PacketType.fromId(dis.readByte());
            
            switch (type) {
                case CACHE_DOWNLOAD_RESPONSE:
                    handleCacheDownloadResponse(dis);
                    break;
                    
                case CACHE_LIST_RESPONSE:
                    handleCacheListResponse(dis);
                    break;
                    
                case CACHE_NOTIFICATION:
                    handleCacheNotification(dis);
                    break;
                    
                default:
                    LOGGER.warn("Unhandled packet type: {}", type);
            }
            
        } catch (Exception e) {
            LOGGER.error("Failed to handle incoming packet", e);
        }
    }
    
    private void handleCacheDownloadResponse(DataInputStream dis) throws IOException {
        String worldName = dis.readUTF();
        String subworldName = dis.readUTF();
        String dimensionName = dis.readUTF();
        String regionKey = dis.readUTF();
        long timestamp = dis.readLong();
        
        CacheMetadata metadata = readMetadata(dis);
        
        int dataLength = dis.readInt();
        byte[] compressedData = new byte[dataLength];
        dis.readFully(compressedData);
        
        String requestKey = buildRequestKey(worldName, subworldName, dimensionName, regionKey);
        CompletableFuture<CacheData> future = pendingRequests.remove(requestKey);
        
        if (future != null) {
            CacheData cacheData = new CacheData(worldName, subworldName, dimensionName,
                                              regionKey, timestamp, UUID.randomUUID(),
                                              compressedData, metadata);
            future.complete(cacheData);
        }
    }
    
    private void sendPacket(byte[] data) {
        // 这里需要根据具体的mod loader实现数据包发送
        // 示例使用Fabric的网络API
        /*
        ClientPlayNetworking.send(
            new Identifier("voxelcache", "main"),
            new PacketByteBuf(Unpooled.wrappedBuffer(data))
        );
        */
    }
    
    private String buildRequestKey(String worldName, String subworldName,
                                 String dimensionName, String regionKey) {
        return String.format("%s/%s/%s/%s", worldName, subworldName, dimensionName, regionKey);
    }
    
    private void writeMetadata(DataOutputStream dos, CacheMetadata metadata) throws IOException {
        dos.writeInt(metadata.getVersion());
        dos.writeLong(metadata.getCreationTime());
        dos.writeInt(metadata.getDataScore());
        dos.writeUTF(metadata.getServerIdentifier());
        
        // 写入区块时间戳
        dos.writeInt(metadata.getChunkTimestamps().size());
        for (Map.Entry<String, Long> entry : metadata.getChunkTimestamps().entrySet()) {
            dos.writeUTF(entry.getKey());
            dos.writeLong(entry.getValue());
        }
    }
    
    private CacheMetadata readMetadata(DataInputStream dis) throws IOException {
        int version = dis.readInt();
        long creationTime = dis.readLong();
        int dataScore = dis.readInt();
        String serverIdentifier = dis.readUTF();
        
        Map<String, Long> chunkTimestamps = new HashMap<>();
        int chunkCount = dis.readInt();
        for (int i = 0; i < chunkCount; i++) {
            String chunkKey = dis.readUTF();
            long timestamp = dis.readLong();
            chunkTimestamps.put(chunkKey, timestamp);
        }
        
        return new CacheMetadata(version, creationTime, chunkTimestamps, 
                               dataScore, serverIdentifier);
    }
}
```

#### 3.3 缓存处理器

```java
// client-patch/CacheProcessor.java
public class CacheProcessor {
    private static final Logger LOGGER = LoggerFactory.getLogger(CacheProcessor.class);
    private final NetworkClient networkClient;
    private final ClientConfig config;
    private final Map<String, Long> lastUploadTime;
    private final Map<String, CacheMetadata> localMetadataCache;
    private final ExecutorService processorExecutor;
    
    public CacheProcessor(NetworkClient networkClient, ClientConfig config) {
        this.networkClient = networkClient;
        this.config = config;
        this.lastUploadTime = new ConcurrentHashMap<>();
        this.localMetadataCache = new ConcurrentHashMap<>();
        this.processorExecutor = Executors.newFixedThreadPool(2);
    }
    
    public void processCacheFile(RegionInfo regionInfo, byte[] fileData, 
                                CacheOperation operation) {
        processorExecutor.execute(() -> {
            try {
                switch (operation) {
                    case UPLOAD:
                        handleUpload(regionInfo, fileData);
                        break;
                        
                    case DELETE:
                        handleDelete(regionInfo);
                        break;
                        
                    default:
                        LOGGER.warn("Unhandled cache operation: {}", operation);
                }
            } catch (Exception e) {
                LOGGER.error("Error processing cache file", e);
            }
        });
    }
    
    private void handleUpload(RegionInfo regionInfo, byte[] fileData) {
        try {
            // 检查上传频率限制
            String cacheKey = buildCacheKey(regionInfo);
            Long lastUpload = lastUploadTime.get(cacheKey);
            
            if (lastUpload != null) {
                long timeSinceLastUpload = System.currentTimeMillis() - lastUpload;
                if (timeSinceLastUpload < config.getUploadCooldown()) {
                    LOGGER.debug("Upload cooldown not met for: {}", cacheKey);
                    return;
                }
            }
            
            // 解析元数据
            CacheMetadata metadata = extractMetadata(fileData);
            
            // 检查是否需要上传
            CacheMetadata localMeta = localMetadataCache.get(cacheKey);
            if (localMeta != null && !shouldUpload(localMeta, metadata)) {
                LOGGER.debug("No need to upload unchanged cache: {}", cacheKey);
                return;
            }
            
            // 创建缓存数据对象
            CacheData cacheData = new CacheData(
                regionInfo.getWorldName(),
                regionInfo.getSubworldName(),
                regionInfo.getDimensionName(),
                regionInfo.getRegionKey(),
                System.currentTimeMillis(),
                getClientUUID(),
                fileData,
                metadata
            );
            
            // 上传数据
            networkClient.uploadCacheData(cacheData).thenAccept(success -> {
                if (success) {
                    lastUploadTime.put(cacheKey, System.currentTimeMillis());
                    localMetadataCache.put(cacheKey, metadata);
                    LOGGER.info("Successfully uploaded cache: {}", cacheKey);
                } else {
                    LOGGER.warn("Failed to upload cache: {}", cacheKey);
                }
            });
            
        } catch (Exception e) {
            LOGGER.error("Error handling cache upload", e);
        }
    }
    
    private void handleDelete(RegionInfo regionInfo) {
        String cacheKey = buildCacheKey(regionInfo);
        lastUploadTime.remove(cacheKey);
        localMetadataCache.remove(cacheKey);
        LOGGER.debug("Removed cache tracking for: {}", cacheKey);
    }
    
    public void checkForUpdates(String worldName, String subworldName, 
                              String dimensionName, int playerX, int playerZ) {
        if (!config.isAutoDownloadEnabled()) {
            return;
        }
        
        processorExecutor.execute(() -> {
            try {
                // 计算需要的区域
                int regionX = playerX >> 8; // 除以256
                int regionZ = playerZ >> 8;
                
                // 检查周围区域
                for (int x = regionX - config.getDownloadRadius(); 
                     x <= regionX + config.getDownloadRadius(); x++) {
                    for (int z = regionZ - config.getDownloadRadius(); 
                         z <= regionZ + config.getDownloadRadius(); z++) {
                        
                        String regionKey = x + "," + z;
                        checkAndDownloadRegion(worldName, subworldName, 
                                             dimensionName, regionKey);
                    }
                }
                
            } catch (Exception e) {
                LOGGER.error("Error checking for updates", e);
            }
        });
    }
    
    private void checkAndDownloadRegion(String worldName, String subworldName,
                                      String dimensionName, String regionKey) {
        Path cacheFile = getCacheFilePath(worldName, subworldName, dimensionName, regionKey);
        
        // 如果本地已有文件，检查是否需要更新
        if (Files.exists(cacheFile)) {
            // 可以添加更新检查逻辑
            return;
        }
        
        // 请求下载
        networkClient.requestCacheData(worldName, subworldName, dimensionName, regionKey)
            .thenAccept(optionalData -> {
                optionalData.ifPresent(cacheData -> {
                    try {
                        // 保存到本地
                        saveCacheToFile(cacheData);
                        LOGGER.info("Downloaded and saved cache: {}/{}/{}/{}", 
                                   worldName, subworldName, dimensionName, regionKey);
                    } catch (Exception e) {
                        LOGGER.error("Failed to save downloaded cache", e);
                    }
                });
            });
    }
    
    private void saveCacheToFile(CacheData cacheData) throws IOException {
        Path cacheFile = getCacheFilePath(cacheData.getWorldName(), 
                                        cacheData.getSubworldName(),
                                        cacheData.getDimensionName(), 
                                        cacheData.getRegionKey());
        
        Files.createDirectories(cacheFile.getParent());
        Files.write(cacheFile, cacheData.getCompressedData(),
                   StandardOpenOption.CREATE, StandardOpenOption.TRUNCATE_EXISTING);
    }
    
    private Path getCacheFilePath(String worldName, String subworldName,
                                String dimensionName, String regionKey) {
        Path minecraftDir = Paths.get(System.getProperty("user.dir"));
        return minecraftDir
            .resolve("voxelmap")
            .resolve("cache")
            .resolve(sanitizeFileName(worldName))
            .resolve(sanitizeFileName(subworldName))
            .resolve(sanitizeFileName(dimensionName))
            .resolve(regionKey + ".zip");
    }
    
    private CacheMetadata extractMetadata(byte[] fileData) {
        // 简化实现，实际需要解析ZIP文件内容
        Map<String, Long> chunkTimestamps = new HashMap<>();
        int dataScore = calculateDataScore(fileData);
        
        return new CacheMetadata(2, System.currentTimeMillis(), chunkTimestamps,
                               dataScore, "client");
    }
    
    private int calculateDataScore(byte[] fileData) {
        // 简单使用文件大小作为数据完整性得分
        return fileData.length;
    }
    
    private boolean shouldUpload(CacheMetadata oldMeta, CacheMetadata newMeta) {
        return newMeta.isNewerThan(oldMeta) || newMeta.isMoreCompleteThan(oldMeta);
    }
    
    private String buildCacheKey(RegionInfo regionInfo) {
        return String.format("%s/%s/%s/%s",
                           regionInfo.getWorldName(),
                           regionInfo.getSubworldName(),
                           regionInfo.getDimensionName(),
                           regionInfo.getRegionKey());
    }
    
    private UUID getClientUUID() {
        // 获取客户端UUID，这里简化处理
        return UUID.randomUUID();
    }
    
    private String sanitizeFileName(String name) {
        if (name == null || name.isEmpty()) {
            return "default";
        }
        return name.replaceAll("[^a-zA-Z0-9._-]", "_");
    }
}
```

#### 3.4 客户端配置

```java
// client-patch/ClientConfig.java
public class ClientConfig {
    private boolean autoUploadEnabled = true;
    private boolean autoDownloadEnabled = true;
    private int uploadCooldown = 30000; // 30秒
    private int downloadRadius = 3; // 下载半径（区域数）
    private int requestTimeout = 10000; // 10秒
    private int maxCacheSize = 100; // MB
    private boolean showNotifications = true;
    
    // Getters and setters...
    public boolean isAutoUploadEnabled() { return autoUploadEnabled; }
    public void setAutoUploadEnabled(boolean enabled) { this.autoUploadEnabled = enabled; }
    
    public boolean isAutoDownloadEnabled() { return autoDownloadEnabled; }
    public void setAutoDownloadEnabled(boolean enabled) { this.autoDownloadEnabled = enabled; }
    
    public int getUploadCooldown() { return uploadCooldown; }
    public void setUploadCooldown(int cooldown) { this.uploadCooldown = cooldown; }
    
    public int getDownloadRadius() { return downloadRadius; }
    public void setDownloadRadius(int radius) { this.downloadRadius = radius; }
    
    public int getRequestTimeout() { return requestTimeout; }
    public void setRequestTimeout(int timeout) { this.requestTimeout = timeout; }
    
    public int getMaxCacheSize() { return maxCacheSize; }
    public void setMaxCacheSize(int size) { this.maxCacheSize = size; }
    
    public boolean isShowNotifications() { return showNotifications; }
    public void setShowNotifications(boolean show) { this.showNotifications = show; }
    
    // 加载和保存配置
    public void load() {
        // 从文件加载配置
    }
    
    public void save() {
        // 保存配置到文件
    }
}
```

### 阶段四：整合与测试

#### 4.1 主客户端类

```java
// client-patch/VoxelMapClient.java
public class VoxelMapClient {
    private static VoxelMapClient instance;
    
    private final ClientConfig config;
    private final NetworkClient networkClient;
    private final CacheProcessor cacheProcessor;
    private final FileWatcher fileWatcher;
    
    private VoxelMapClient() {
        this.config = new ClientConfig();
        this.config.load();
        
        this.networkClient = new NetworkClient(config);
        this.cacheProcessor = new CacheProcessor(networkClient, config);
        this.fileWatcher = new FileWatcher(networkClient, cacheProcessor);
    }
    
    public static VoxelMapClient getInstance() {
        if (instance == null) {
            instance = new VoxelMapClient();
        }
        return instance;
    }
    
    public void initialize() {
        // 启动文件监控
        fileWatcher.start();
        
        // 注册事件监听器
        registerEventListeners();
        
        LOGGER.info("VoxelMap Cache Client initialized");
    }
    
    public void shutdown() {
        fileWatcher.stop();
        config.save();
        
        LOGGER.info("VoxelMap Cache Client shut down");
    }
    
    private void registerEventListeners() {
        // 注册玩家移动事件，检查是否需要下载新区域
        // 注册世界加载事件，初始化缓存
        // 等等...
    }
    
    public ClientConfig getConfig() {
        return config;
    }
    
    public NetworkClient getNetworkClient() {
        return networkClient;
    }
    
    public CacheProcessor getCacheProcessor() {
        return cacheProcessor;
    }
}
```

## 部署说明

### Velocity插件部署

1. 编译插件：
```bash
cd velocity-plugin
mvn clean package
```

2. 将生成的JAR文件复制到Velocity的plugins目录

3. 重启Velocity服务器

### 客户端补丁部署

1. 编译客户端补丁：
```bash
cd client-patch
./gradlew build
```

2. 将生成的mod文件添加到客户端mods目录

3. 确保VoxelMap已安装

## 配置说明

### Velocity配置 (config.yml)

```yaml
voxelmap-cache-share:
  # 缓存存储设置
  storage:
    max-cache-size: 10240 # MB
    retention-days: 30
    compression-level: 6
  
  # 网络设置
  network:
    max-packet-size: 1048576 # 1MB
    timeout: 10000 # ms
  
  # 冲突解决设置
  conflict-resolution:
    strategy: "intelligent" # timestamp, version, intelligent
    merge-threshold: 300000 # 5分钟
```

### 客户端配置 (voxelcache.json)

```json
{
  "autoUpload": true,
  "autoDownload": true,
  "uploadCooldown": 30000,
  "downloadRadius": 3,
  "requestTimeout": 10000,
  "maxCacheSize": 100,
  "showNotifications": true
}
```

## 性能优化建议

1. **文件监控优化**：
   - 使用去抖动技术避免频繁上传
   - 批量处理文件变化

2. **网络传输优化**：
   - 使用压缩减少传输数据量
   - 实现增量更新而非全量替换

3. **内存管理**：
   - 限制缓存大小
   - 定期清理过期数据

4. **并发处理**：
   - 使用线程池管理并发任务
   - 实现请求合并减少重复请求

## 未来扩展

1. **Web界面**：提供Web界面查看和管理缓存数据

2. **权限系统**：添加权限控制，限制上传/下载权限

3. **统计功能**：记录缓存使用统计信息

4. **备份功能**：定期备份重要缓存数据

5. **API接口**：提供REST API供其他工具使用

## 故障排除

### 常见问题

1. **上传失败**：
   - 检查网络连接
   - 确认Velocity插件已启动
   - 查看客户端日志

2. **下载失败**：
   - 检查服务器是否有对应缓存
   - 确认客户端配置正确

3. **文件监控不工作**：
   - 确认VoxelMap缓存目录正确
   - 检查文件系统权限

### 日志位置

- Velocity插件日志：`velocity/logs/`
- 客户端日志：`.minecraft/logs/`

## 结语

本系统通过巧妙地利用VoxelMap的缓存机制，实现了地图数据的共享功能。整个实现不需要修改VoxelMap原始代码，具有良好的兼容性和可维护性。