## Thành Phần Chính

Cơ chế này cần các thành phần sau:

```text
1. Bộ phân loại gói MOVE/FIRE.
2. Bộ nhớ đệm để giữ gói khi bật Dịch Chuyển.
3. Cơ chế phát lại khi tắt Dịch Chuyển.
4. Cơ chế bù MOVE khi đang phát lại.
5. Hàm gửi lại gói UDP ra máy chủ.
```

## Hằng Số Cần Có

Đặt các hằng số này trong lớp service:

```java
private static final int DROP_MIN = 0x15;
private static final int DROP_MAX = 0x1c2;
private static final int TELE_BUF_MAX = 10000;

private static final int TELE_PACKET_NONE = 0;
private static final int TELE_PACKET_MOVE = 1;
private static final int TELE_PACKET_FIRE = 2;

private static final int TELE_MOVE_MIN = 0x37;
private static final int TELE_MOVE_MAX = 0xa0;

private static final int TELE_FIRE_MIN = DROP_MIN;
private static final int TELE_FIRE_MAX = DROP_MAX;
private static final int TELE_BROAD_FIRE_MAX = 0x36;

private static final int TELE_REPLAY_DELAY_DEFAULT_MS = 1;
private static final int TELE_REPLAY_DELAY_MIN_MS = 1;
private static final int TELE_REPLAY_DELAY_MAX_MS = 50;
```

Ý nghĩa:

```text
TELE_PACKET_NONE: gói không liên quan đến Dịch Chuyển.
TELE_PACKET_MOVE: gói di chuyển.
TELE_PACKET_FIRE: gói bắn.
TELE_BUF_MAX: số gói tối đa được giữ.
TELE_REPLAY_DELAY_DEFAULT_MS: độ trễ mặc định giữa các gói khi phát lại.
```

## Dữ Liệu Gói Được Giữ

Mỗi gói được giữ cần lưu:

```text
phần dữ liệu UDP
IP đích
port đích
port nguồn
loại gói MOVE/FIRE
```

Code:

```java
static final class _TP {
    final byte[] data;
    final int dip;
    final int dp;
    final int sp;
    final int type;

    _TP(byte[] data, int dip, int dp, int sp, int type) {
        this.data = data;
        this.dip = dip;
        this.dp = dp;
        this.sp = sp;
        this.type = type;
    }
}
```

## Biến Trạng Thái Cần Có

```java
private volatile boolean isJitter = false;
private volatile boolean _teleReplaying = false;

private volatile int teleReplayDelayMs = TELE_REPLAY_DELAY_DEFAULT_MS;

private final ConcurrentLinkedQueue<_TP> _teleBuf = new ConcurrentLinkedQueue<>();

private final Object _teleCatchupLock = new Object();
private final ArrayList<_TP> _teleCatchupBuf = new ArrayList<_TP>();
```

Ý nghĩa:

```text
isJitter:
  true khi Dịch Chuyển đang bật.

_teleReplaying:
  true khi dịch vụ đang xả lại gói đã giữ.

_teleBuf:
  bộ nhớ đệm chính, giữ MOVE/FIRE trong lúc Dịch Chuyển bật.

_teleCatchupBuf:
  bộ nhớ đệm phụ, giữ MOVE mới phát sinh trong lúc phát lại.
```

## Hàm Hỗ Trợ Cơ Bản

```java
private static byte[] copyPayload(byte[] src, int off, int len) {
    byte[] copy = new byte[len];
    System.arraycopy(src, off, copy, 0, len);
    return copy;
}

private static int ub(byte[] data, int index) {
    return data[index] & 0xff;
}

private static boolean hasBytes(byte[] data, int off, int len, int rel, int b0, int b1) {
    return off >= 0
            && len >= rel + 2
            && off + rel + 1 < data.length
            && ub(data, off + rel) == b0
            && ub(data, off + rel + 1) == b1;
}

private static boolean hasBytes(byte[] data, int off, int len, int rel, int b0, int b1, int b2) {
    return off >= 0
            && len >= rel + 3
            && off + rel + 2 < data.length
            && ub(data, off + rel) == b0
            && ub(data, off + rel + 1) == b1
            && ub(data, off + rel + 2) == b2;
}

private static String teleTypeName(int type) {
    if (type == TELE_PACKET_MOVE) return "MOVE";
    return "FIRE";
}

private static String teleNote(int type, int queueSize) {
    return "type=" + teleTypeName(type) + " queue=" + queueSize;
}
```

## Phân Loại Gói MOVE Và FIRE

Đây là phần quyết định gói nào bị giữ lại.

```java
private static int classifyTelePacket(byte[] data, int off, int payloadLen) {
    if (isTeleMovePayload(payloadLen)) {
        return TELE_PACKET_MOVE;
    }
    if (isTeleFirePayload(data, off, payloadLen)) {
        return TELE_PACKET_FIRE;
    }
    return TELE_PACKET_NONE;
}

private static boolean isTeleMovePayload(int payloadLen) {
    return payloadLen >= TELE_MOVE_MIN && payloadLen <= TELE_MOVE_MAX;
}

private static boolean isBroadTeleFirePayload(int payloadLen) {
    return !isTeleMovePayload(payloadLen)
            && payloadLen >= TELE_FIRE_MIN
            && payloadLen <= TELE_BROAD_FIRE_MAX;
}

private static boolean isTeleFirePayload(byte[] data, int off, int payloadLen) {
    if (isMaskedTeleFirePayload(data, off, payloadLen)) return true;
    if (isCurrentTeleFirePayload(data, off, payloadLen)) return true;
    return isLegacyTeleFirePayload(data, off, payloadLen);
}
```

Giải thích:

```text
MOVE:
  Được nhận diện chủ yếu theo kích thước payload.

FIRE:
  Được nhận diện theo pattern byte.

NONE:
  Gói không liên quan đến cơ chế Dịch Chuyển.
```

## Nhận Diện FIRE

```java
private static boolean isCurrentTeleFirePayload(byte[] data, int off, int payloadLen) {
    if (off < 0 || off + payloadLen > data.length || payloadLen < 10) return false;

    if (payloadLen == 34) {
        if (!hasBytes(data, off, payloadLen, 2, 0x81, 0x98, 0x80)) return false;
        if (ub(data, off + 5) != 0x82 || ub(data, off + 7) != 0x80 || ub(data, off + 9) != 0x80) {
            return false;
        }
        int marker = ub(data, off + 8);
        return marker == 0xE8 || marker == 0xE9 || marker == 0x06;
    }

    if (payloadLen == 26) {
        return hasBytes(data, off, payloadLen, 2, 0x81, 0x90, 0x80)
                && ub(data, off + 5) == 0x82
                && ub(data, off + 7) == 0x80
                && ub(data, off + 8) == 0xEC
                && ub(data, off + 9) == 0x80;
    }

    return false;
}
```

## Nhận Diện FIRE Dạng Masked

```java
private static boolean isMaskedTeleFirePayload(byte[] data, int off, int payloadLen) {
    if (off < 0 || off + payloadLen > data.length || payloadLen < 10) return false;

    if (payloadLen == 34) {
        return ub(data, off + 2) == 0x58
                && ub(data, off + 4) == 0x5A
                && ub(data, off + 6) == 0x58
                && ub(data, off + 7) == 0x5B
                && ub(data, off + 8) == 0x42
                && ub(data, off + 9) == 0x5A;
    }

    if (payloadLen == 26) {
        return ub(data, off + 2) == 0x58
                && ub(data, off + 4) == 0x5A
                && ub(data, off + 6) == 0x58
                && ub(data, off + 7) == 0x5B
                && ub(data, off + 8) == 0x4A
                && ub(data, off + 9) == 0x5A;
    }

    return false;
}
```

## Nhận Diện FIRE Phụ

Một số gói FIRE phụ có kích thước 24 byte. Chúng cũng cần được giữ khi Dịch Chuyển đang bật hoặc đang phát lại.

```java
private static boolean isTeleFireAuxPayload(byte[] data, int off, int payloadLen) {
    return isMaskedTeleFireAuxPayload(data, off, payloadLen)
            || isCurrentTeleFireAuxPayload(data, off, payloadLen)
            || isLegacyTeleFireAuxPayload(data, off, payloadLen);
}

private static boolean isCurrentTeleFireAuxPayload(byte[] data, int off, int payloadLen) {
    return payloadLen == 24
            && hasBytes(data, off, payloadLen, 2, 0x81, 0x90, 0x80)
            && ub(data, off + 5) == 0x80
            && ub(data, off + 6) == 0x82
            && ub(data, off + 7) == 0x80;
}

private static boolean isLegacyTeleFireAuxPayload(byte[] data, int off, int payloadLen) {
    return payloadLen == 24
            && hasBytes(data, off, payloadLen, 2, 0xBB, 0xB9, 0xBB)
            && hasBytes(data, off, payloadLen, 5, 0xBA, 0xAB, 0xBB);
}

private static boolean isMaskedTeleFireAuxPayload(byte[] data, int off, int payloadLen) {
    return off >= 0
            && off + payloadLen <= data.length
            && payloadLen == 24
            && hasBytes(data, off, payloadLen, 2, 0x5A, 0x58, 0x5A)
            && ub(data, off + 5) == 0x5B
            && ub(data, off + 6) == 0x4A
            && ub(data, off + 7) == 0x5A;
}
```

## Khi Nào Bắt FIRE Phụ

```java
private boolean isTeleFireCaptureActive() {
    return isJitter || _teleReplaying;
}
```

Khi đang bật Dịch Chuyển hoặc đang phát lại, nếu gói chưa được nhận diện là FIRE nhưng khớp FIRE phụ, thì ép loại thành FIRE:

```java
int telePacketType = classifyTelePacket(buf, off, plen);

if (telePacketType == TELE_PACKET_NONE
        && isFilterEnabled
        && isTeleFireCaptureActive()
        && (isTeleFireAuxPayload(buf, off, plen) || isBroadTeleFirePayload(plen))) {
    telePacketType = TELE_PACKET_FIRE;
}
```

## Bật Dịch Chuyển

Khi người dùng bật nút `Dịch Chuyển`, cần xóa bộ nhớ đệm cũ rồi bật trạng thái giữ gói.

```java
private void startTele() {
    _teleBuf.clear();
    clearTeleCatchupBuffer();
    _teleReplaying = false;
    isJitter = true;
    logDevControl("MODE_TELE_ON", "mode=tele");
}
```

Nếu dùng nút bật/tắt chung:

```java
private void toggleTele() {
    if (isJitter) {
        stopTeleAndReplay();
    } else {
        startTele();
    }
}
```

## Tắt Dịch Chuyển Và Phát Lại

Khi người dùng tắt `Dịch Chuyển`, lấy toàn bộ gói đã giữ rồi phát lại.

```java
private void stopTeleAndReplay() {
    isJitter = false;

    if (_teleBuf.isEmpty()) {
        _teleReplaying = false;
        clearTeleCatchupBuffer();
        logDevControl("MODE_TELE_OFF_EMPTY", "mode=tele queue=0");
        return;
    }

    logDevControl("MODE_TELE_OFF_REPLAY", "mode=tele queue=" + _teleBuf.size());
    _teleReplaying = true;
    startTeleReplay();
}

private void startTeleReplay() {
    if (_teleBuf.isEmpty()) return;

    _teleReplaying = true;

    final List<_TP> all = new ArrayList<_TP>();
    _TP tp;
    while ((tp = _teleBuf.poll()) != null) {
        all.add(tp);
    }

    threadPool.execute(new TeleReplayRunnable(all));
}
```

## Giữ Gói Trong Vòng Lặp UDP

Đặt đoạn này sau khi đã phân tích phần dữ liệu UDP thành công, trước khi gửi gói trực tiếp ra máy chủ.

Biến cần có sẵn:

```text
buf  = bộ đệm gói
off  = vị trí bắt đầu phần dữ liệu UDP
plen = độ dài phần dữ liệu UDP
sp   = cổng nguồn
dp   = cổng đích
dip  = IPv4 đích dạng int
```

Code:

```java
int telePacketType = classifyTelePacket(buf, off, plen);

if (telePacketType == TELE_PACKET_NONE
        && isFilterEnabled
        && isTeleFireCaptureActive()
        && (isTeleFireAuxPayload(buf, off, plen) || isBroadTeleFirePayload(plen))) {
    telePacketType = TELE_PACKET_FIRE;
}

if (isFilterEnabled && isJitter && telePacketType != TELE_PACKET_NONE) {
    if (_teleBuf.size() < TELE_BUF_MAX) {
        _teleBuf.offer(new _TP(copyPayload(buf, off, plen), dip, dp, sp, telePacketType));
        logDevPacket("DELAY_TELE_" + teleTypeName(telePacketType), "OUT",
                buf, off, plen, sp, dp, dip, teleNote(telePacketType, _teleBuf.size()));
    } else {
        logDevPacket("DROP_TELE_BUFFER_FULL_" + teleTypeName(telePacketType), "OUT",
                buf, off, plen, sp, dp, dip, teleNote(telePacketType, _teleBuf.size()));
    }
    continue;
}
```

Ý nghĩa:

```text
Nếu Dịch Chuyển đang bật:
  - MOVE/FIRE được sao chép vào _teleBuf.
  - Gói không được gửi trực tiếp.
  - Thứ tự gói trong _teleBuf là thứ tự gốc lúc trò chơi gửi.
```

## Xử Lý Gói Khi Đang Phát Lại

Trong lúc phát lại, người chơi có thể vẫn di chuyển hoặc bắn.

Cơ chế hiện tại:

```text
MOVE mới phát sinh trong lúc phát lại:
  đưa vào bộ nhớ đệm bù để phát lại sau.

FIRE mới phát sinh trong lúc phát lại:
  bỏ, không gửi trực tiếp, không đưa vào bộ nhớ đệm.
```

Code:

```java
if (isFilterEnabled && _teleReplaying && telePacketType != TELE_PACKET_NONE) {
    if (telePacketType == TELE_PACKET_MOVE) {
        int queueSize = offerTeleCatchup(new _TP(copyPayload(buf, off, plen), dip, dp, sp, telePacketType));
        logDevPacket("CATCHUP_TELE_" + teleTypeName(telePacketType), "OUT",
                buf, off, plen, sp, dp, dip, teleNote(telePacketType, queueSize));
    } else {
        logDevPacket("DROP_LIVE_DURING_REPLAY_TELE_" + teleTypeName(telePacketType), "OUT",
                buf, off, plen, sp, dp, dip,
                "type=" + teleTypeName(telePacketType) + " replay=tele");
    }
    continue;
}
```

## Bù MOVE

```java
private int offerTeleCatchup(_TP tp) {
    synchronized (_teleCatchupLock) {
        while (_teleCatchupBuf.size() >= TELE_BUF_MAX) {
            _teleCatchupBuf.remove(0);
        }
        _teleCatchupBuf.add(tp);
        return _teleCatchupBuf.size();
    }
}

private List<_TP> drainTeleCatchupBuffer() {
    synchronized (_teleCatchupLock) {
        if (_teleCatchupBuf.isEmpty()) return new ArrayList<_TP>();
        List<_TP> all = new ArrayList<_TP>(_teleCatchupBuf);
        _teleCatchupBuf.clear();
        return all;
    }
}

private void clearTeleCatchupBuffer() {
    synchronized (_teleCatchupLock) {
        _teleCatchupBuf.clear();
    }
}
```

## Luồng Phát Lại

```java
private class TeleReplayRunnable implements Runnable {
    private final List<_TP> all;

    TeleReplayRunnable(List<_TP> all) {
        this.all = all;
    }

    @Override
    public void run() {
        try {
            replayTelePackets(all);

            while (true) {
                List<_TP> catchup = drainTeleCatchupBuffer();
                if (catchup.isEmpty()) break;
                replayPacketList(catchup, "REPLAY_TELE_CATCHUP_", "REPLAY_CATCHUP_SEND_FAIL_", teleReplayDelayMs);
            }
        } finally {
            _teleReplaying = false;
            _teleBuf.clear();
            clearTeleCatchupBuffer();
        }
    }
}

private void replayTelePackets(List<_TP> all) {
    replayPacketList(all, "REPLAY_TELE_", "REPLAY_SEND_FAIL_", teleReplayDelayMs);
}
```

Điểm quan trọng:

```text
Danh sách all được phát lại đúng thứ tự.
Không tách MOVE và FIRE.
Không lọc bớt FIRE.
Không gom gói.
```

## Gửi Lại Gói Đã Giữ

Hàm này gửi lại phần dữ liệu UDP đã giữ lên máy chủ.

Đoạn mã dưới đây dùng `udpMap` để tìm lại socket cũ. Nếu không tìm thấy socket cũ, nó tạo socket dự phòng để gửi.

```java
private void replayPacketList(List<_TP> all, String replayPrefix, String failPrefix, int replayDelay) {
    int safeDelay = replayDelay < 0 ? 0 : replayDelay;

    for (int index = 0; index < all.size(); index++) {
        _TP tp = all.get(index);

        if (Thread.currentThread().isInterrupted() || !isRun || !isActive || !isFilterEnabled) {
            logDevPacket(replayPrefix + "ABORT_" + teleTypeName(tp.type), "OUT",
                    tp.data, 0, tp.data.length, tp.sp, tp.dp, tp.dip,
                    "type=" + teleTypeName(tp.type) + " replay_aborted");
            break;
        }

        long key = ((long)(tp.sp & 0xffff) << 48)
                | ((long)(tp.dp & 0xffff) << 32)
                | (tp.dip & 0xFFFFFFFFL);

        SE se = udpMap.get(key);
        boolean sent = false;

        if (se != null) {
            try {
                DatagramPacket p = new DatagramPacket(
                        tp.data,
                        0,
                        tp.data.length,
                        InetAddress.getByAddress(tb(tp.dip)),
                        tp.dp
                );
                se.s.send(p);
                se.touch();
                sent = true;

                logDevPacket(replayPrefix + teleTypeName(tp.type), "OUT",
                        tp.data, 0, tp.data.length, tp.sp, tp.dp, tp.dip,
                        "type=" + teleTypeName(tp.type) + " delay_ms=" + safeDelay);
            } catch (Exception e) {

            }
        }

        if (!sent) {
            DatagramSocket fallback = null;
            try {
                fallback = new DatagramSocket();
                protect(fallback);

                DatagramPacket p = new DatagramPacket(
                        tp.data,
                        0,
                        tp.data.length,
                        InetAddress.getByAddress(tb(tp.dip)),
                        tp.dp
                );
                fallback.send(p);
                sent = true;

                logDevPacket(replayPrefix + "FALLBACK_" + teleTypeName(tp.type), "OUT",
                        tp.data, 0, tp.data.length, tp.sp, tp.dp, tp.dip,
                        "type=" + teleTypeName(tp.type) + " delay_ms=" + safeDelay);
            } catch (Exception e) {
                logDevPacket(failPrefix + teleTypeName(tp.type), "OUT",
                        tp.data, 0, tp.data.length, tp.sp, tp.dp, tp.dip,
                        "type=" + teleTypeName(tp.type) + " fallback_error");
            } finally {
                if (fallback != null) {
                    try { fallback.close(); } catch (Exception e) { }
                }
            }
        }

        if (safeDelay > 0) {
            try { Thread.sleep(safeDelay); } catch (Exception e) { }
        }
    }
}
```

## Phiên Socket Cần Có

Nếu mã nguồn chưa có lớp phiên UDP, có thể dùng mẫu này:

```java
static final class SE {
    final DatagramSocket s;
    volatile long t;

    SE(DatagramSocket socket) {
        s = socket;
        t = System.currentTimeMillis();
    }

    void touch() {
        t = System.currentTimeMillis();
    }
}
```

Bảng phiên:

```java
private final ConcurrentHashMap<Long, SE> udpMap = new ConcurrentHashMap<>();
```

Hàm chuyển IP int sang byte:

```java
private static byte[] tb(int ip) {
    return new byte[] {
            (byte)(ip >> 24),
            (byte)(ip >> 16),
            (byte)(ip >> 8),
            (byte)ip
    };
}
```

## Cách Tích Hợp

Thứ tự chuyển cơ chế:

```text
1. Thêm hằng số.
2. Thêm lớp _TP.
3. Thêm biến trạng thái isJitter, _teleReplaying.
4. Thêm _teleBuf và _teleCatchupBuf.
5. Thêm các hàm phân loại MOVE/FIRE.
6. Thêm startTele, stopTeleAndReplay, startTeleReplay.
7. Thêm TeleReplayRunnable.
8. Thêm replayPacketList.
9. Chèn cơ chế giữ gói vào vòng lặp UDP gửi ra.
10. Gắn nút bật/tắt vào toggleTele.
```

Vị trí quan trọng nhất:

```text
Cơ chế giữ gói phải nằm sau bước phân tích phần dữ liệu UDP.
Cơ chế giữ gói phải nằm trước đoạn gửi trực tiếp bằng DatagramPacket.
```

## Lỗi Thường Gặp

### Bắn Không Tính Dame

Nguyên nhân thường gặp:

```text
Phát lại sai thứ tự MOVE/FIRE.
FIRE bị bỏ bớt.
FIRE bị gửi quá nhanh.
FIRE phụ không được nhận diện.
```

Cách xử lý:

```text
Giữ nguyên thứ tự gốc.
Không tách MOVE và FIRE.
Không gom rút gọn FIRE.
Tăng teleReplayDelayMs lên 5ms hoặc 10ms nếu máy chủ bỏ gói.
```

### Bị Kéo Về Vị Trí Cũ Khi Vẫn Đang Bật Dịch Chuyển

Đây là phản hồi vị trí từ máy chủ.

Khi Dịch Chuyển đang bật:

```text
Máy khách chạy từ A đến B.
Máy chủ chưa nhận MOVE nên vẫn nghĩ người chơi ở A.
Máy chủ gửi trạng thái ép máy khách về A.
```

Khi tắt Dịch Chuyển:

```text
MOVE được phát lại.
Máy chủ nhận đường đi đến B.
Máy khách bị kéo lại về B.
```

Đây là đánh đổi tự nhiên của kiểu giữ MOVE lâu. Muốn giảm hiện tượng này thì phải xả MOVE định kỳ, nhưng như vậy Dịch Chuyển sẽ yếu hơn.

### Mất Kết Nối

Nguyên nhân thường gặp:

```text
Bắt quá rộng gói không phải MOVE/FIRE.
Giữ nhầm gói quan trọng của trò chơi.
Phát lại quá nhanh.
Chặn gói sửa vị trí từ máy chủ quá mạnh.
```

Cách xử lý:

```text
Chỉ giữ gói được phân loại là MOVE/FIRE.
Không bắt toàn bộ gói 21..450.
Không chặn gói sửa vị trí từ máy chủ.
Không dùng độ trễ phát lại quá thấp nếu máy chủ skip hoặc decline gói.
```
