# Java基础-操作文件
{docsify-updated}

- [Java基础-操作文件](#java基础-操作文件)
  - [创建文件](#创建文件)
  - [重命名或者移动文件](#重命名或者移动文件)
  - [删除文件](#删除文件)
  - [监控文件系统 `WatchService`](#监控文件系统-watchservice)


### 创建文件
1. 使用 NIO 的 `Files` 类： `Files.createFile(Path newFilePath)`
   ```
   @Test
    public void givenUsingNio_whenCreatingFile_thenCorrect() throws IOException {
        Path newFilePath = Paths.get(FILE_NAME);
        Files.createFile(newFilePath);
    }
   ```
   值得注意的是，新的 API 很好地利用了异常。如果文件已经存在，我们不再需要检查返回代码。相反，我们将获得 `FileAlreadyExistsException` 异常：

2. `File.createNewFile()`
    ```
    public void givenUsingFile_whenCreatingFile_thenCorrect() throws IOException {
        File newFile = new File(FILE_NAME);
        boolean success = newFile.createNewFile();
        assertTrue(success);
    }
    ```
    请注意，文件必须不存在，此操作才会成功。如果文件确实存在，则 `createNewFile()` 操作将返回 false。

3. 使用 `FileOutputStream`
   ```
   @Test
    public void givenUsingFileOutputStream_whenCreatingFile_thenCorrect() throws IOException {
        try(FileOutputStream fileOutputStream = new FileOutputStream(FILE_NAME)){
        }
    }
   ```
   在这种情况下，当我们实例化 `FileOutputStream` 对象时，就会创建一个新文件。如果给定名称的文件已经存在，它将被覆盖。但是，如果现文件是一个目录，或者由于某种原因无法创建新文件，那么我们就会收到 `FileNotFoundException` 异常。

### 重命名或者移动文件
```
Path fileToMovePath = Paths.get(FILE_TO_MOVE);
Path targetPath = Paths.get(TARGET_FILE);
Files.move(fileToMovePath, targetPath);

File fileToMove = new File(FILE_TO_MOVE);
boolean isMoved = fileToMove.renameTo(new File(TARGET_FILE));
if (!isMoved) {
    throw new FileSystemException(TARGET_FILE);
}
```

### 删除文件
```
File fileToDelete = new File("src/test/resources/fileToDelete_jdk6.txt");
boolean success = fileToDelete.delete();

Path fileToDeletePath = Paths.get("src/test/resources/fileToDelete_jdk7.txt");
Files.delete(fileToDeletePath);
```

### 监控文件系统 `WatchService`
```
WatchService watchService
                = FileSystems.getDefault().newWatchService();

Path path = Paths.get(System.getProperty("user.home"));

path.register(
    watchService,
    new WatchEvent.Kind[] {StandardWatchEventKinds.ENTRY_CREATE,
            StandardWatchEventKinds.ENTRY_DELETE,
            StandardWatchEventKinds.ENTRY_MODIFY
    },
    SensitivityWatchEventModifier.HIGH);

WatchKey key;
while ((key = watchService.take()) != null) {
    for (WatchEvent<?> event : key.pollEvents()) {
        System.out.println(
                "Event kind:" + event.kind()
                        + ". File affected: " + event.context() + ".");
    }
    key.reset();
}
```
