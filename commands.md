### Datei erzeugen und in HDFS ablegen

Textdatei anlegen, neues Verzeichnis im HDFS erstellen und die Datei dahin kopieren (geht auch mit -copyFromLocal, aber -put ist kürzer):

````
[root@sandbox hadoop]# echo "Dies ist die Textdatei fuer BigData." > test.txt
[root@sandbox hadoop]# hdfs dfs -ls
Found 3 items
drwx------   - root hdfs          0 2016-11-03 16:21 .staging
drwxr-xr-x   - root hdfs          0 2016-11-03 16:21 out1
-rw-r--r--   1 root hdfs     249321 2016-11-03 16:17 pg14591.txt
[root@sandbox hadoop]# hdfs dfs -mkdir test
[root@sandbox hadoop]# hdfs dfs -put test.txt test
[root@sandbox hadoop]# hdfs dfs -ls test
Found 1 items
-rw-r--r--   1 root hdfs         37 2016-11-03 20:15 test/test.txt
```

Dateigröße anzeigen:
```
[root@sandbox hadoop]# hdfs dfs -du -h test/test.txt
37  test/test.txt
```

Mit dem Parameter -h wird die Dateigröße in einer für Menschen besser lesbaren Form dargestellt. Das macht aber bei einer Datei, die nur 37 Byte groß ist, keinen Unterschied.

### Zwei Dateien per HDFS mergen

```
[root@sandbox hadoop]# echo "Dies ist der zweite Teil der Datei." > test2.txt
[root@sandbox hadoop]# hdfs dfs -put test2.txt test/test2.txt
[root@sandbox hadoop]# hdfs dfs -ls test
Found 2 items
-rw-r--r--   1 root hdfs         37 2016-11-03 20:15 test/test.txt
-rw-r--r--   1 root hdfs         36 2016-11-03 20:26 test/test2.txt
[root@sandbox hadoop]# hdfs dfs -du test
37  test/test.txt
36  test/test2.txt
```

Beide Dateien in eine mergen und auf dem lokalen Filesystem ablegen:
[root@sandbox hadoop]# hdfs dfs -getmerge test merged_file.txt

### Backup und Recovery mit Snapshots

Snapshot des Verzeichnis *test* erstellen:

```
[root@sandbox hadoop]# hdfs dfs -createSnapshot test snapshot01
16/11/03 20:36:33 WARN retry.RetryInvocationHandler: Exception while invoking ClientNamenodeProtocolTranslatorPB.createSnapshot over null. Not retrying because try once and fail.
org.apache.hadoop.ipc.RemoteException(org.apache.hadoop.hdfs.protocol.SnapshotException): Directory is not a snapshottable directory: /user/root/test
...
createSnapshot: Directory is not a snapshottable directory: /user/root/test
```

Das Verzeichnis muss erst "snapshottable" gemacht werden:

```
[root@sandbox hadoop]# hdfs dfsadmin -allowSnapshot test
16/11/03 20:37:37 WARN retry.RetryInvocationHandler: Exception while invoking ClientNamenodeProtocolTranslatorPB.allowSnapshot over null. Not retrying because try once and fail.
org.apache.hadoop.ipc.RemoteException(org.apache.hadoop.security.AccessControlException): Access denied for user root. Superuser privilege is required
```

Access denied --> ab hier nehmen wir den Superuser hdfs, der ausreichend Rechte hat.

```
[root@sandbox hadoop]# su hdfs
[hdfs@sandbox hadoop]$ hdfs dfs -mkdir /snaptest
[hdfs@sandbox hadoop]$ hdfs dfs -ls /snaptest
[hdfs@sandbox hadoop]$ hdfs dfs -createSnapshot /snaptest snap01
16/11/03 20:54:25 WARN retry.RetryInvocationHandler: Exception while invoking ClientNamenodeProtocolTranslatorPB.createSnapshot over null. Not retrying because try once and fail.
org.apache.hadoop.ipc.RemoteException(org.apache.hadoop.hdfs.protocol.SnapshotException): Directory is not a snapshottable directory: /snaptest
createSnapshot: Directory is not a snapshottable directory: /snaptest
[hdfs@sandbox hadoop]$ hdfs dfsadmin -allowSnapshot /snaptest
Allowing snaphot on /snaptest succeeded
```

Datei für den Test erzeugen, im HDFS ablegen und Snapshot erstellen:

```
[hdfs@sandbox hadoop]$ hdfs dfs -put merged_file.txt /snaptest
[hdfs@sandbox hadoop]$ hdfs dfs -createSnapshot /snaptest snap01
Created snapshot /snaptest/.snapshot/snap01
```

Inhalt des Snapshots anzeigen:
```
[hdfs@sandbox hadoop]$ hdfs dfs -ls /snaptest/.snapshot/snap01
Found 1 items
-rw-r--r--   1 hdfs hdfs         73 2016-11-03 21:03 /snaptest/.snapshot/snap01/merged_file.txt
```

Datei hinzufügen, neuen Snapshot erzeugen und die beiden Snapshots vergleichen:

```
[hdfs@sandbox hadoop]$ hdfs dfs -put test.txt /snaptest
[hdfs@sandbox hadoop]$ hdfs dfs -createSnapshot /snaptest snap02
Created snapshot /snaptest/.snapshot/snap02
[hdfs@sandbox hadoop]$ hdfs snapshotDiff /snaptest snap01 snap02
Difference between snapshot snap01 and snapshot snap02 under directory /snaptest:
M       .
+       ./test.txt
```

Verzeichnis *snaptest* löschen:
```
[hdfs@sandbox hadoop]$ hdfs dfs -rm -r /snaptest
…
rm: Failed to move to trash: hdfs://sandbox.hortonworks.com:8020/snaptest: The directory /snaptest cannot be deleted since /snaptest is snapshottable and already has snapshots
```

Das Verzeichnis kann wegen der Snapshots nicht gelöscht werden.

Datei test.txt aus dem Verzeichnis snaptest löschen und mithilfe des Snapshots wiederherstellen:

```
[hdfs@sandbox hadoop]$ hdfs dfs -rm /snaptest/test.txt
16/11/03 21:17:27 INFO fs.TrashPolicyDefault: Moved: 'hdfs://sandbox.hortonworks.com:8020/snaptest/test.txt' to trash at: hdfs://sandbox.hortonworks.com:8020/user/hdfs/.Trash/Current/snaptest/test.txt
[hdfs@sandbox hadoop]$ hdfs dfs -ls /snaptest
Found 1 items
-rw-r--r--   1 hdfs hdfs         73 2016-11-03 21:03 /snaptest/merged_file.txt
[hdfs@sandbox hadoop]$ hdfs dfs -cp /snaptest/.snapshot/snap02/test.txt /snaptest
[hdfs@sandbox hadoop]$ hdfs dfs -ls /snaptest
Found 2 items
-rw-r--r--   1 hdfs hdfs         73 2016-11-03 21:03 /snaptest/merged_file.txt

-rw-r--r--   1 hdfs hdfs         37 2016-11-03 21:18 /snaptest/test.txt
```
