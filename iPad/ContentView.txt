// Pencil Bridge v0.0.9
import SwiftUI
import UIKit
import Network
import ImageIO
import AVFoundation
import CoreMedia

struct PenEvent: Encodable {
    let type: String
    let x: Double
    let y: Double
    let p: Double
    var source: String = "pencil"
    static let release = PenEvent(type: "release", x: 0, y: 0, p: 0)
}

final class BridgeModel: ObservableObject {
    @Published var status = "شروع USB را بزن و کد را در ویندوز وارد کن."
    @Published var pairingCode = ""
    @Published var listening = false
    @Published var connected = false
    @Published var enabled = false
    @Published var toolbarActivity = Date()
    @Published var clearID = 0
    private(set) var samples = 0
    @Published var acknowledged = 0
    private(set) var pressure = 0.0
    @Published var screenImage: UIImage?
    @Published var screenAspect: CGFloat = 16.0 / 9.0
    private var screenFrames = 0
    @Published var videoReady = false
    let videoView = PencilPadView()
    private let videoAssembler = H264Assembler()
    private var videoGeneration = 0
    private var lastResync = Date.distantPast
    private(set) var touchSamples = 0
    @Published var resetViewID = 0
    @Published var frameMs = 0
    @Published var fps = 0
    @Published var smoothing = 0.75
    @Published var pressureSensitivity = 1.0
    private var fpsStarted = Date()
    private var fpsFrames = 0
    private var lastFrame = Date.distantPast
    private let decodeQueue = DispatchQueue(label: "PencilBridge.h264", qos: .userInteractive)
    private var listener: NWListener?
    private var peer: NWConnection?
    private var incoming = Data()
    private var pending: [PenEvent] = []
    private var busy = false
    private var timer: Timer?
    private var lastPoint: PenEvent?
    private var heartbeat = Date.distantPast

    init() {
        pairingCode = String(format: "%02d", Int.random(in: 0...99))
        let heartbeatTimer = Timer(timeInterval: 1.0 / 60.0, repeats: true) { [weak self] _ in self?.tick() }
        timer = heartbeatTimer
        RunLoop.main.add(heartbeatTimer, forMode: .common)
    }
    deinit { timer?.invalidate() }

    func startUSB() {
        guard listener == nil else { return }
        do {
            let port = NWEndpoint.Port(rawValue: 9876)!
            let parameters = NWParameters.tcp
            let server = try NWListener(using: parameters, on: port)
            listener = server
            status = "در حال آماده‌سازی USB…"
            server.stateUpdateHandler = { [weak self, weak server] state in
                guard let self = self, let server = server, self.listener === server else { return }
                switch state {
                case .ready:
                    self.listening = true
                    self.status = "آماده؛ start-usb.cmd را در ویندوز اجرا و کد بالا را وارد کن."
                case .waiting(let error):
                    self.status = "Listener waiting: " + error.localizedDescription
                case .failed(let error):
                    self.status = "Listener failed: " + error.localizedDescription
                    self.stopUSB(keepStatus: true)
                default: break
                }
            }
            server.newConnectionHandler = { [weak self] connection in
                self?.accept(connection)
            }
            server.start(queue: .main)
        } catch { status = "Cannot start USB listener: " + error.localizedDescription }
    }

    private func accept(_ connection: NWConnection) {
        guard peer == nil else { connection.cancel(); return }
        peer = connection
        incoming.removeAll()
        connection.stateUpdateHandler = { [weak self, weak connection] state in
            guard let self = self, let connection = connection, self.peer === connection else { return }
            switch state {
            case .ready: self.read(connection)
            case .failed(let error): self.dropPeer("USB failed: " + error.localizedDescription)
            case .cancelled: self.dropPeer("اتصال قطع شد؛ ویندوز خودکار دوباره تلاش می‌کند.")
            default: break
            }
        }
        connection.start(queue: .main)
        DispatchQueue.main.asyncAfter(deadline: .now() + 8) { [weak self, weak connection] in
            guard let self = self, let connection = connection, self.peer === connection, !self.connected else { return }
            self.dropPeer("کد اتصال از ویندوز دریافت نشد؛ دوباره تلاش کن.")
        }
    }

    private func read(_ connection: NWConnection) {
        connection.receive(minimumIncompleteLength: 1, maximumLength: 65536) { [weak self, weak connection] data, _, complete, error in
            guard let self = self, let connection = connection, self.peer === connection else { return }
            if let data = data { self.incoming.append(data) }
            if self.incoming.count > 20 * 1024 * 1024 { self.dropPeer("پیام نامعتبر است."); return }
            while !self.incoming.isEmpty {
                if self.incoming.count >= 4 && self.incoming.prefix(4) == Data("PBHV".utf8) {
                    guard self.connected else { self.dropPeer("Frame before pairing"); return }
                    if self.incoming.count < 12 { break }
                    let header = Array(self.incoming.prefix(12))
                    let length = header[4..<8].reduce(0) { ($0 << 8) | Int($1) }
                    let id = header[8..<12].reduce(UInt32(0)) { ($0 << 8) | UInt32($1) }
                    if length < 4 || length > 4 * 1024 * 1024 { self.dropPeer("Invalid JPEG size"); return }
                    if self.incoming.count < 12 + length { break }
                    let jpeg = Data(self.incoming.dropFirst(12).prefix(length))
                    self.incoming.removeFirst(12 + length)
                    self.decodeVideo(jpeg, id: id, connection: connection)
                    continue
                }
                if self.incoming.count < 4 { break }
                guard let index = self.incoming.firstIndex(of: 10) else { break }
                let line = Data(self.incoming[..<index])
                self.incoming.removeSubrange(...index)
                if line.isEmpty { continue }
                guard let json = (try? JSONSerialization.jsonObject(with: line)) as? [String: Any] else {
                    self.dropPeer("پیام ویندوز قابل خواندن نیست."); return
                }
                if !self.connected {
                    guard json["type"] as? String == "hello", json["code"] as? String == self.pairingCode else {
                        self.dropPeer("کد اتصال اشتباه است."); return
                    }
                    guard let version = json["version"] as? String, ["0.0.4", "0.0.5", "0.0.6", "0.0.7", "0.0.8", "0.0.9"].contains(version) else {
                        self.dropPeer("نسخه ویندوز هم باید v0.0.9 باشد."); return
                    }
                    self.connected = true
                    self.fpsStarted = Date(); self.fpsFrames = 0; self.fps = 0
                    self.acknowledged = 0
                    self.status = "USB متصل شد؛ منتظر تصویر ویندوز…"
                    connection.send(content: Data("{\"type\":\"ready\",\"version\":\"0.0.9\"}\n".utf8), completion: .contentProcessed { [weak self] error in
                        if let error = error { self?.dropPeer(error.localizedDescription) }
                    })
                } else if json["type"] as? String == "videoReset" {
                    self.resetVideo()
                } else if json["type"] as? String == "ack", let count = json["count"] as? Int {
                    self.acknowledged = count
                } else if json["type"] as? String == "frame",
                          let jpeg = json["jpeg"] as? String,
                          let bytes = Data(base64Encoded: jpeg),
                          let image = UIImage(data: bytes) {
                    self.screenImage = image
                    self.screenAspect = image.size.width / max(1, image.size.height)
                    self.recordFrame()
                    if self.screenFrames == 1 { self.status = "تصویر ویندوز رسید؛ کنترل را فعال کن." }
                } else if json["type"] as? String == "metrics", let ms = json["frameMs"] as? Int {
                    self.frameMs = ms
                } else if json["type"] as? String == "screenError" {
                    self.status = json["message"] as? String ?? "Screen capture failed."
                } else if json["type"] as? String == "screenStopped" {
                    self.pauseWriting(); self.screenImage = nil; self.screenFrames = 0; self.resetVideo()
                    self.status = "اشتراک تصویر متوقف شد؛ برنامه ویندوز را دوباره اجرا کن."
                } else { self.dropPeer("پیام ناشناخته از ویندوز."); return }
            }
            if let error = error { self.dropPeer(error.localizedDescription); return }
            if complete { self.dropPeer("کابل یا برنامه ویندوز قطع شد."); return }
            self.read(connection)
        }
    }

    private func resetVideo() {
        videoGeneration += 1; videoReady = false; videoView.flushVideo()
        decodeQueue.async { [weak self] in self?.videoAssembler.reset() }
    }
    func suspendInput() {
        videoView.cancelInput(); pauseWriting()
    }
    func resumeVideo() {
        if listener == nil { startUSB() }
        if connected { resetVideo(); requestResync() }
    }
    private func requestResync() {
        guard connected, let connection = peer, Date().timeIntervalSince(lastResync) > 0.25 else { return }
        lastResync = Date(); pauseWriting()
        connection.send(content: Data("{\"type\":\"videoResync\"}\n".utf8), completion: .contentProcessed { _ in })
    }
    private func decodeVideo(_ bytes: Data, id: UInt32, connection: NWConnection) {
        let generation = videoGeneration
        decodeQueue.async { [weak self, weak connection] in
            guard let self = self else { return }
            do {
                let sample = try self.videoAssembler.sample(bytes, id: id)
                DispatchQueue.main.async { [weak self, weak connection] in
                    guard let self = self, let connection = connection,
                          self.peer === connection, self.videoGeneration == generation else { return }
                    if let sample = sample {
                        if self.videoView.enqueueVideo(sample) {
                            if !self.videoReady {
                                self.videoReady = true
                                if let format = CMSampleBufferGetFormatDescription(sample) {
                                    let size = CMVideoFormatDescriptionGetDimensions(format)
                                    self.screenAspect = CGFloat(size.width) / max(1, CGFloat(size.height))
                                }
                                self.status = "تصویر H.264 رسید؛ کنترل را فعال کن."
                            }
                            self.recordFrame()
                        } else { self.requestResync() }
                    } else { self.requestResync() }
                    let ack = "{\"type\":\"frameAck\",\"id\":\(id)}\n"
                    connection.send(content: Data(ack.utf8), completion: .contentProcessed { _ in })
                }
            } catch {
                DispatchQueue.main.async { [weak self, weak connection] in
                    guard let self = self, let connection = connection, self.peer === connection else { return }
                    self.status = error.localizedDescription; self.requestResync()
                    let ack = "{\"type\":\"frameAck\",\"id\":\(id)}\n"
                    connection.send(content: Data(ack.utf8), completion: .contentProcessed { _ in })
                }
            }
        }
    }

    private func recordFrame() {
        screenFrames += 1; fpsFrames += 1; lastFrame = Date()
        let duration = lastFrame.timeIntervalSince(fpsStarted)
        if duration >= 1 { fps = Int((Double(fpsFrames) / duration).rounded()); fpsFrames = 0; fpsStarted = lastFrame }
    }

    func receive(_ event: PenEvent) {
        if event.source == "touch" { touchSamples += 1 } else { samples += 1; pressure = event.p }
        guard enabled, connected else { return }
        if pending.count >= 512 {
            pending = [.release]; enabled = false; lastPoint = nil
            status = "ارسال عقب افتاده؛ متوقف شد. دوباره شروع کن."
            flush(); return
        }
        pending.append(event)
        lastPoint = (event.type == "down" || event.type == "move") ? event : nil
        if event.type != "move" { flush() }
    }

    func togglePen() {
        videoView.cancelInput()
        if enabled { pauseWriting() }
        else if connected && videoReady && fps > 0 { enabled = true }
        toolbarActivity = Date()
    }

    func pauseWriting() {
        enabled = false; lastPoint = nil
        if connected { pending.append(.release); flush() }
    }

    func stopUSB(keepStatus: Bool = false) {
        pauseWriting()
        listener?.stateUpdateHandler = nil
        listener?.newConnectionHandler = nil
        listener?.cancel(); listener = nil; listening = false
        dropPeer(keepStatus ? status : "USB متوقف شد.")
    }

    private func tick() {
        if Date().timeIntervalSince(lastFrame) > 2 { fps = 0; if enabled { pauseWriting() } }
        if enabled, let point = lastPoint, Date().timeIntervalSince(heartbeat) > 0.35 {
            heartbeat = Date()
            pending.append(PenEvent(type: "move", x: point.x, y: point.y, p: point.p, source: point.source))
        }
        flush()
    }

    private func flush() {
        guard connected, !busy, !pending.isEmpty, let connection = peer else { return }
        let batch = Array(pending.prefix(128)); pending.removeFirst(batch.count)
        var bytes = Data()
        for event in batch {
            guard let data = try? JSONEncoder().encode(event) else { continue }
            bytes.append(data); bytes.append(10)
        }
        busy = true
        connection.send(content: bytes, completion: .contentProcessed { [weak self, weak connection] error in
            guard let self = self, let connection = connection, self.peer === connection else { return }
            self.busy = false
            if let error = error { self.dropPeer(error.localizedDescription) } else { self.flush() }
        })
    }

    private func dropPeer(_ message: String) {
        peer?.stateUpdateHandler = nil
        peer?.cancel(); peer = nil
        connected = false; enabled = false; busy = false
        lastPoint = nil; pending.removeAll(); incoming.removeAll()
        status = message
        resetVideo(); screenImage = nil; screenFrames = 0; fps = 0; fpsFrames = 0; fpsStarted = Date()
    }
}

final class H264Assembler {
    private var sps: Data?
    private var pps: Data?
    private var format: CMVideoFormatDescription?
    func reset() { sps = nil; pps = nil; format = nil }
    private func units(_ data: Data) -> [Data] {
        let b = [UInt8](data)
        var starts: [(Int, Int)] = []
        var i = 0
        while i + 3 <= b.count {
            if b[i] == 0 && b[i + 1] == 0 {
                if b[i + 2] == 1 { starts.append((i, 3)); i += 3; continue }
                if i + 4 <= b.count && b[i + 2] == 0 && b[i + 3] == 1 { starts.append((i, 4)); i += 4; continue }
            }
            i += 1
        }
        return starts.enumerated().compactMap { index, start in
            let end = index + 1 < starts.count ? starts[index + 1].0 : b.count
            let begin = start.0 + start.1
            return begin < end ? Data(b[begin..<end]) : nil
        }
    }
    func sample(_ bytes: Data, id: UInt32) throws -> CMSampleBuffer? {
        let nals = units(bytes)
        var changed = false
        for nal in nals {
            guard let first = nal.first else { continue }
            if first & 31 == 7 && sps != nal { sps = nal; changed = true }
            if first & 31 == 8 && pps != nal { pps = nal; changed = true }
        }
        if changed { format = nil }
        if format == nil, let sps = sps, let pps = pps {
            var created: CMFormatDescription?
            let status = sps.withUnsafeBytes { s in
                pps.withUnsafeBytes { p in
                    let pointers = [s.bindMemory(to: UInt8.self).baseAddress!, p.bindMemory(to: UInt8.self).baseAddress!]
                    let sizes = [sps.count, pps.count]
                    return pointers.withUnsafeBufferPointer { pointerBuffer in
                        sizes.withUnsafeBufferPointer { sizeBuffer in
                            CMVideoFormatDescriptionCreateFromH264ParameterSets(allocator: kCFAllocatorDefault,
                                parameterSetCount: 2, parameterSetPointers: pointerBuffer.baseAddress!,
                                parameterSetSizes: sizeBuffer.baseAddress!, nalUnitHeaderLength: 4, formatDescriptionOut: &created)
                        }
                    }
                }
            }
            guard status == noErr, let created = created else { throw problem("H264 format", status) }
            format = created
        }
        guard let format = format else { return nil }
        var avcc = Data()
        var hasSlice = false
        for nal in nals {
            guard let first = nal.first else { continue }
            let type = first & 31
            if type == 1 || type == 5 { hasSlice = true }
            if type == 7 || type == 8 || type == 9 { continue }
            var length = UInt32(nal.count).bigEndian
            withUnsafeBytes(of: &length) { avcc.append(contentsOf: $0) }
            avcc.append(nal)
        }
        guard hasSlice else { return nil }
        var block: CMBlockBuffer?
        var status = CMBlockBufferCreateWithMemoryBlock(allocator: kCFAllocatorDefault, memoryBlock: nil,
            blockLength: avcc.count, blockAllocator: kCFAllocatorDefault, customBlockSource: nil,
            offsetToData: 0, dataLength: avcc.count, flags: 0, blockBufferOut: &block)
        guard status == noErr, let block = block else { throw problem("H264 block", status) }
        status = avcc.withUnsafeBytes { bytes in
            CMBlockBufferReplaceDataBytes(with: bytes.baseAddress!, blockBuffer: block, offsetIntoDestination: 0, dataLength: avcc.count)
        }
        guard status == noErr else { throw problem("H264 copy", status) }
        var timing = CMSampleTimingInfo(duration: CMTime(value: 1, timescale: 60),
            presentationTimeStamp: CMTime(value: Int64(id), timescale: 60), decodeTimeStamp: .invalid)
        var size = avcc.count
        var sample: CMSampleBuffer?
        status = CMSampleBufferCreateReady(allocator: kCFAllocatorDefault, dataBuffer: block,
            formatDescription: format, sampleCount: 1, sampleTimingEntryCount: 1,
            sampleTimingArray: &timing, sampleSizeEntryCount: 1, sampleSizeArray: &size, sampleBufferOut: &sample)
        guard status == noErr, let sample = sample else { throw problem("H264 sample", status) }
        if let attachments = CMSampleBufferGetSampleAttachmentsArray(sample, createIfNecessary: true) {
            let dictionary = unsafeBitCast(CFArrayGetValueAtIndex(attachments, 0), to: CFMutableDictionary.self)
            CFDictionarySetValue(dictionary, Unmanaged.passUnretained(kCMSampleAttachmentKey_DisplayImmediately).toOpaque(),
                Unmanaged.passUnretained(kCFBooleanTrue).toOpaque())
            if !nals.contains(where: { ($0.first ?? 0) & 31 == 5 }) {
                CFDictionarySetValue(dictionary, Unmanaged.passUnretained(kCMSampleAttachmentKey_NotSync).toOpaque(),
                    Unmanaged.passUnretained(kCFBooleanTrue).toOpaque())
            }
        }
        return sample
    }
    private func problem(_ message: String, _ code: OSStatus) -> NSError {
        NSError(domain: "PencilBridge.H264", code: Int(code), userInfo: [NSLocalizedDescriptionKey: message + " (\(code))"])
    }
}

// Directional stabilizer: stronger lateral filtering, corner response and bounded lag.
struct PencilMotionFilter {
    private var raw: CGPoint?
    private var output: CGPoint?
    private var timestamp: TimeInterval?
    private var velocity = CGPoint.zero
    mutating func reset() { raw = nil; output = nil; timestamp = nil; velocity = .zero }
    private func alpha(_ cutoff: Double, _ dt: Double) -> CGFloat {
        CGFloat(1 / (1 + 1 / (2 * Double.pi * cutoff * dt)))
    }
    mutating func sample(_ point: CGPoint, time: TimeInterval, strength: Double) -> CGPoint {
        guard let previousRaw = raw, let previousOutput = output, let previousTime = timestamp,
              time - previousTime <= 0.08 else {
            raw = point; output = point; timestamp = time; velocity = .zero; return point
        }
        guard time > previousTime else { return previousOutput }
        let dt = min(0.05, max(0.001, time - previousTime))
        let amount = min(1, max(0, strength))
        let derivativeWeight = alpha(1.5, dt)
        velocity.x += derivativeWeight * ((point.x - previousRaw.x) / CGFloat(dt) - velocity.x)
        velocity.y += derivativeWeight * ((point.y - previousRaw.y) / CGFloat(dt) - velocity.y)
        let speed = Double(hypot(velocity.x, velocity.y))
        let error = CGPoint(x: point.x - previousOutput.x, y: point.y - previousOutput.y)
        var result: CGPoint
        if amount == 0 { result = point }
        else if speed > 15 {
            let ux = velocity.x / CGFloat(speed), uy = velocity.y / CGFloat(speed)
            let along = error.x * ux + error.y * uy
            let across = -error.x * uy + error.y * ux
            let forwardWeight = alpha(18 - 10 * amount + 0.06 * speed, dt)
            var lateralWeight = alpha(8 - 7.2 * amount + 0.0015 * speed, dt)
            let turn = CGFloat(min(1, max(0, (Double(abs(across)) - (2 + 2 * amount)) / 3)))
            lateralWeight += (forwardWeight - lateralWeight) * turn
            result = CGPoint(x: previousOutput.x + ux * along * forwardWeight - uy * across * lateralWeight,
                             y: previousOutput.y + uy * along * forwardWeight + ux * across * lateralWeight)
        } else {
            let weight = alpha(12 - 10 * amount, dt)
            result = CGPoint(x: previousOutput.x + error.x * weight, y: previousOutput.y + error.y * weight)
        }
        let lag = hypot(point.x - result.x, point.y - result.y)
        let limit = CGFloat(3 + speed * 0.018)
        if lag > limit {
            result = CGPoint(x: point.x + (result.x - point.x) * limit / lag,
                             y: point.y + (result.y - point.y) * limit / lag)
        }
        raw = point; output = result; timestamp = time
        return result
    }
}

final class PencilPadView: UIView, UIGestureRecognizerDelegate, UIPencilInteractionDelegate {
    var onPencilDoubleTap: (() -> Void)?
    private var lastDoubleTap = Date.distantPast
    var onEvent: ((PenEvent) -> Void)?
    var remoteImage: UIImage? { didSet { setNeedsDisplay() } }
    private let videoLayer = AVSampleBufferDisplayLayer()
    var videoAvailable = false
    var smoothing = 0.75
    var pressureSensitivity = 1.0
    private var motionFilter = PencilMotionFilter()
    private var filteredPressure = 0.0
    private var pencilEnded = Date.distantPast
    func enqueueVideo(_ sample: CMSampleBuffer) -> Bool {
        if #available(iOS 17.0, *) {
            let renderer = videoLayer.sampleBufferRenderer
            guard renderer.status != .failed, renderer.isReadyForMoreMediaData else { return false }
            renderer.enqueue(sample)
        } else {
            guard videoLayer.status != .failed, videoLayer.isReadyForMoreMediaData else { return false }
            videoLayer.enqueue(sample)
        }
        videoAvailable = true
        return true
    }
    func flushVideo() {
        if #available(iOS 17.0, *) {
            videoLayer.sampleBufferRenderer.flush(removingDisplayedImage: true, completionHandler: nil)
        } else {
            videoLayer.flushAndRemoveImage()
        }
        videoAvailable = false; setNeedsDisplay()
    }
    private func updateVideoTransform() {
        CATransaction.begin(); CATransaction.setDisableActions(true)
        videoLayer.anchorPoint = .zero; videoLayer.bounds = CGRect(origin: .zero, size: bounds.size)
        videoLayer.position = offset; videoLayer.transform = CATransform3DMakeScale(zoom, zoom, 1)
        CATransaction.commit()
    }
    var sending = false {
        didSet {
            if oldValue != sending {
                activeTouch = nil; previous = nil; fingerDragging = false
            }
        }
    }
    private var activeTouch: UITouch?
    private var previous: CGPoint?
    private var zoom: CGFloat = 1
    private var offset = CGPoint.zero
    private var lastSize = CGSize.zero
    private var fingerDragging = false
    private var fingerPoint = CGPoint.zero

    override init(frame: CGRect) {
        super.init(frame: frame)
        let pencilInteraction = UIPencilInteraction()
        pencilInteraction.delegate = self
        addInteraction(pencilInteraction)
        backgroundColor = UIColor(red: 0.98, green: 0.97, blue: 0.94, alpha: 1)
        isMultipleTouchEnabled = true
        clipsToBounds = true
        videoLayer.videoGravity = .resize
        layer.addSublayer(videoLayer)
        let tap = UITapGestureRecognizer(target: self, action: #selector(tapped(_:)))
        let doubleTap = UITapGestureRecognizer(target: self, action: #selector(doubleTapped(_:)))
        doubleTap.numberOfTapsRequired = 2
        tap.require(toFail: doubleTap)
        let pinch = UIPinchGestureRecognizer(target: self, action: #selector(pinched(_:)))
        let pan = UIPanGestureRecognizer(target: self, action: #selector(panned(_:)))
        pan.minimumNumberOfTouches = 2; pan.maximumNumberOfTouches = 2
        let hold = UILongPressGestureRecognizer(target: self, action: #selector(held(_:)))
        hold.minimumPressDuration = 0.35
        tap.require(toFail: hold)
        doubleTap.require(toFail: hold)
        hold.numberOfTouchesRequired = 1
        for gesture in [tap, doubleTap, pinch, pan, hold] as [UIGestureRecognizer] {
            gesture.allowedTouchTypes = [NSNumber(value: UITouch.TouchType.direct.rawValue)]
            gesture.cancelsTouchesInView = false
            gesture.requiresExclusiveTouchType = true
            gesture.delegate = self
            addGestureRecognizer(gesture)
        }
    }
    private func handlePencilDoubleTap() {
        guard Date().timeIntervalSince(lastDoubleTap) > 0.3 else { return }
        lastDoubleTap = Date(); cancelInput(); onPencilDoubleTap?()
    }
    @available(iOS, introduced: 12.1, deprecated: 17.5)
    func pencilInteractionDidTap(_ interaction: UIPencilInteraction) { handlePencilDoubleTap() }
    @available(iOS 17.5, *)
    func pencilInteraction(_ interaction: UIPencilInteraction, didReceiveTap tap: UIPencilInteraction.Tap) { handlePencilDoubleTap() }
    required init?(coder: NSCoder) { fatalError("init(coder:) is not supported") }
    func clear() { cancelInput() }
    func resetView() { zoom = 1; offset = .zero; updateVideoTransform(); setNeedsDisplay() }
    override func layoutSubviews() {
        super.layoutSubviews()
        updateVideoTransform()
        if lastSize != bounds.size { lastSize = bounds.size; cancelStroke(); endFingerDrag(); resetView() }
    }
    func gestureRecognizer(_ gestureRecognizer: UIGestureRecognizer, shouldRecognizeSimultaneouslyWith otherGestureRecognizer: UIGestureRecognizer) -> Bool {
        return (gestureRecognizer is UIPinchGestureRecognizer && otherGestureRecognizer is UIPanGestureRecognizer)
            || (gestureRecognizer is UIPanGestureRecognizer && otherGestureRecognizer is UIPinchGestureRecognizer)
    }
    func gestureRecognizer(_ gestureRecognizer: UIGestureRecognizer, shouldReceive touch: UITouch) -> Bool {
        touch.type == .direct && activeTouch == nil && Date().timeIntervalSince(pencilEnded) > 0.25
    }
    func cancelInput() { cancelStroke(); endFingerDrag() }
    private func clampOffset() {
        offset.x = min(0, max(bounds.width * (1 - zoom), offset.x))
        offset.y = min(0, max(bounds.height * (1 - zoom), offset.y))
    }
    private func imagePoint(_ point: CGPoint) -> CGPoint {
        CGPoint(x: (point.x - offset.x) / zoom, y: (point.y - offset.y) / zoom)
    }
    @objc private func pinched(_ gesture: UIPinchGestureRecognizer) {
        guard activeTouch == nil, !fingerDragging else { return }
        let location = gesture.location(in: self)
        let anchor = imagePoint(location)
        zoom = min(5, max(1, zoom * gesture.scale))
        offset = CGPoint(x: location.x - anchor.x * zoom, y: location.y - anchor.y * zoom)
        gesture.scale = 1; clampOffset(); updateVideoTransform(); setNeedsDisplay()
    }
    @objc private func panned(_ gesture: UIPanGestureRecognizer) {
        guard activeTouch == nil, !fingerDragging else { return }
        let delta = gesture.translation(in: self)
        offset.x += delta.x; offset.y += delta.y
        gesture.setTranslation(.zero, in: self); clampOffset(); updateVideoTransform(); setNeedsDisplay()
    }
    @objc private func tapped(_ gesture: UITapGestureRecognizer) {
        click(gesture.location(in: self), count: 1)
    }
    @objc private func doubleTapped(_ gesture: UITapGestureRecognizer) {
        click(gesture.location(in: self), count: 2)
    }
    private func click(_ point: CGPoint, count: Int) {
        guard sending, (videoAvailable || remoteImage != nil), activeTouch == nil, !fingerDragging else { return }
        for _ in 0..<count {
            emitPoint(point, type: "down", pressure: 0.5, source: "touch")
            emitPoint(point, type: "up", pressure: 0, source: "touch")
        }
    }
    @objc private func held(_ gesture: UILongPressGestureRecognizer) {
        guard activeTouch == nil else { return }
        let point = gesture.location(in: self)
        switch gesture.state {
        case .began:
            guard sending, (videoAvailable || remoteImage != nil) else { return }
            fingerDragging = true; fingerPoint = point
            emitPoint(point, type: "down", pressure: 0.5, source: "touch")
        case .changed:
            if fingerDragging { fingerPoint = point; emitPoint(point, type: "move", pressure: 0.5, source: "touch") }
        case .ended, .cancelled, .failed: endFingerDrag()
        default: break
        }
    }
    private func endFingerDrag() {
        if fingerDragging { emitPoint(fingerPoint, type: "up", pressure: 0, source: "touch") }
        fingerDragging = false
    }
    override func draw(_ rect: CGRect) {}
    override func touchesBegan(_ touches: Set<UITouch>, with event: UIEvent?) {
        guard sending, videoAvailable, activeTouch == nil,
              let touch = touches.first(where: { $0.type == .pencil }) else { return }
        endFingerDrag()
        for recognizer in gestureRecognizers ?? [] { recognizer.isEnabled = false; recognizer.isEnabled = true }
        motionFilter.reset(); filteredPressure = 0
        activeTouch = touch; previous = imagePoint(touch.location(in: self))
        emit(touch, type: "down")

    }
    override func touchesMoved(_ touches: Set<UITouch>, with event: UIEvent?) {
        guard let active = activeTouch, touches.contains(where: { $0 === active }) else { return }
        for touch in event?.coalescedTouches(for: active) ?? [active] { emit(touch, type: "move") }
    }
    override func touchesEnded(_ touches: Set<UITouch>, with event: UIEvent?) {
        guard let active = activeTouch, touches.contains(where: { $0 === active }) else { return }
        for sample in event?.coalescedTouches(for: active) ?? [] { emit(sample, type: "move") }
        emit(active, type: "up"); activeTouch = nil; previous = nil; motionFilter.reset(); pencilEnded = Date()
    }
    override func touchesCancelled(_ touches: Set<UITouch>, with event: UIEvent?) {
        if let active = activeTouch, touches.contains(where: { $0 === active }) { cancelStroke() }
    }
    private func cancelStroke() {
        if activeTouch != nil { onEvent?(.release) }
        activeTouch = nil; previous = nil; motionFilter.reset(); pencilEnded = Date()
    }
    private func force(_ touch: UITouch) -> Double {
        guard touch.maximumPossibleForce > 0 else { return 0.5 }
        let raw = min(1, max(0, Double(touch.force / touch.maximumPossibleForce)))
        return pow(raw, 1 / pressureSensitivity)
    }
    private func emit(_ touch: UITouch, type: String) {
        let point = motionFilter.sample(touch.location(in: self), time: touch.timestamp, strength: smoothing)
        let measured = force(touch)
        filteredPressure = type == "down" ? measured : filteredPressure * 0.25 + measured * 0.75
        emitPoint(point, type: type, pressure: type == "up" ? 0 : filteredPressure, source: "pencil")
    }
    private func emitPoint(_ location: CGPoint, type: String, pressure: Double, source: String) {
        guard bounds.width > 0, bounds.height > 0 else { return }
        let point = imagePoint(location)
        onEvent?(PenEvent(type: type,
            x: min(1, max(0, Double(point.x / bounds.width))),
            y: min(1, max(0, Double(point.y / bounds.height))), p: pressure, source: source))
    }
}

struct PencilPad: UIViewRepresentable {
    @ObservedObject var model: BridgeModel
    class Coordinator { var clearID = 0; var resetID = 0 }
    func makeCoordinator() -> Coordinator { Coordinator() }
    func makeUIView(context: Context) -> PencilPadView {
        let view = model.videoView
        view.onEvent = { [weak model = model] event in model?.receive(event) }
        view.onPencilDoubleTap = { [weak model = model] in model?.togglePen() }
        return view
    }
    func updateUIView(_ view: PencilPadView, context: Context) {
        view.sending = model.enabled
        view.smoothing = model.smoothing
        view.pressureSensitivity = model.pressureSensitivity
        view.remoteImage = model.screenImage
        if context.coordinator.clearID != model.clearID { context.coordinator.clearID = model.clearID; view.clear() }
        if context.coordinator.resetID != model.resetViewID { context.coordinator.resetID = model.resetViewID; view.resetView() }
    }
}

private let learkSky = Color(red: 0.16, green: 0.57, blue: 0.88)

struct LearkButton: View {
    let title: String
    let icon: String
    var prominent = false
    let action: () -> Void
    @ViewBuilder var body: some View {
        #if compiler(>=6.2)
        if #available(iOS 26.0, *) {
            if prominent {
                Button(action: action) { label }.buttonStyle(.glassProminent).tint(learkSky)
            } else {
                Button(action: action) { label }.buttonStyle(.glass).tint(learkSky)
            }
        } else { legacy }
        #else
        legacy
        #endif
    }
    private var label: some View {
        Label(title, systemImage: icon).font(.subheadline.weight(.semibold)).padding(.vertical, 5)
    }
    private var legacy: some View {
        Button(action: action) { label }.buttonStyle(LearkLegacyButtonStyle(prominent: prominent))
    }
}

struct LearkLegacyButtonStyle: ButtonStyle {
    var prominent: Bool
    @Environment(\.accessibilityReduceMotion) private var reduceMotion
    func makeBody(configuration: Configuration) -> some View {
        configuration.label.padding(.horizontal, 16).padding(.vertical, 8)
            .foregroundStyle(prominent ? Color.white : learkSky)
            .background(.ultraThinMaterial, in: Capsule())
            .background(prominent ? learkSky : Color.clear, in: Capsule())
            .overlay(Capsule().strokeBorder(learkSky.opacity(0.25), lineWidth: 1))
            .scaleEffect(configuration.isPressed && !reduceMotion ? 0.96 : 1)
            .animation(reduceMotion ? nil : .spring(response: 0.3, dampingFraction: 0.7), value: configuration.isPressed)
    }
}

struct LearkConnectionMark: View {
    let waiting: Bool
    let connected: Bool
    @Environment(\.accessibilityReduceMotion) private var reduceMotion
    var body: some View {
        TimelineView(.animation(minimumInterval: 1.0 / 30.0, paused: !waiting || reduceMotion)) { timeline in
            let phase = waiting && !reduceMotion ? timeline.date.timeIntervalSinceReferenceDate : 0
            ZStack {
                Circle().fill(learkSky.opacity(0.08)).frame(width: 148, height: 148)
                Circle().stroke(learkSky.opacity(0.15), lineWidth: 1).frame(width: 130, height: 130)
                Circle().trim(from: 0, to: waiting ? 0.24 : 1)
                    .stroke(learkSky.opacity(waiting ? 0.8 : 0.3), style: StrokeStyle(lineWidth: 3, lineCap: .round))
                    .frame(width: 130, height: 130).rotationEffect(.degrees(phase * 90))
                Image(systemName: connected ? "checkmark" : "pencil.tip")
                    .font(.system(size: 44, weight: .light)).foregroundStyle(learkSky)
                    .scaleEffect(waiting && !reduceMotion ? 1 + 0.025 * sin(phase * 3) : 1)
            }
        }.frame(height: 160).accessibilityHidden(true)
    }
}

struct ContentView: View {
    @StateObject private var model = BridgeModel()
    @State private var writing = false
    @State private var showSettings = false
    @State private var toolbarVisible = true
    @State private var toolbarHide: DispatchWorkItem?
    @State private var splashStarted = false
    @State private var splashVisible = true
    @State private var splashReveal = false
    @State private var splashAccent = false
    @State private var splashExit = false
    @AppStorage("learkpen.language") private var language = "en"
    @AppStorage("learkpen.languageChosen") private var languageChosen = false
    @AppStorage("learkpen.appearance") private var appearance = "system"
    @Environment(\.colorScheme) private var colorScheme
    @Environment(\.accessibilityReduceMotion) private var reduceMotion
    private var selectedScheme: ColorScheme? {
        appearance == "dark" ? .dark : appearance == "light" ? .light : nil
    }
    private var dark: Bool { selectedScheme == .dark || (selectedScheme == nil && colorScheme == .dark) }
    private var background: Color { dark ? .black : .white }
    var body: some View {
        ZStack {
            background.ignoresSafeArea()
            if writing { desktop }
            else { connectionPage }
            if splashVisible { splash }
            else if !languageChosen { languageWelcome }
        }
        .environment(\.layoutDirection, language == "fa" ? .rightToLeft : .leftToRight)
        .onAppear {
            guard !splashStarted else { return }
            splashStarted = true
            withAnimation(reduceMotion ? nil : .easeOut(duration: 0.65)) { splashReveal = true }
            DispatchQueue.main.asyncAfter(deadline: .now() + 0.22) {
                withAnimation(reduceMotion ? nil : .easeInOut(duration: 0.55)) { splashAccent = true }
            }
            DispatchQueue.main.asyncAfter(deadline: .now() + 2.26) {
                withAnimation(reduceMotion ? nil : .easeOut(duration: 0.24)) { splashExit = true }
            }
            DispatchQueue.main.asyncAfter(deadline: .now() + 2.5) { splashVisible = false }
        }
        .onReceive(model.$toolbarActivity) { _ in revealToolbar() }
        .onDisappear { toolbarHide?.cancel() }
        .tint(learkSky)
        .preferredColorScheme(selectedScheme)
        .sheet(isPresented: $showSettings, onDismiss: { revealToolbar() }) { settings }
        .statusBarHidden(writing)
        .onReceive(model.$connected) { if $0 { writing = true; revealToolbar() } }
        .onReceive(NotificationCenter.default.publisher(for: UIApplication.willResignActiveNotification)) { _ in model.suspendInput() }
        .onReceive(NotificationCenter.default.publisher(for: UIApplication.didBecomeActiveNotification)) { _ in model.resumeVideo() }
    }
    private func tr(_ text: String) -> String {
        if language == "fa" { return text }
        let translations: [String: String] = [
            "شروع USB را بزن و کد را در ویندوز وارد کن.": "Start the connection, then enter the code on Windows.",
            "در حال آماده‌سازی USB…": "Preparing USB…",
            "آماده؛ start-usb.cmd را در ویندوز اجرا و کد بالا را وارد کن.": "Ready. Run start-usb.cmd on Windows and enter the code above.",
            "USB متصل شد؛ منتظر تصویر ویندوز…": "USB connected. Waiting for the Windows display…",
            "تصویر ویندوز رسید؛ کنترل را فعال کن.": "Display received. Enable pen control.",
            "تصویر H.264 رسید؛ کنترل را فعال کن.": "Live display ready. Enable pen control.",
            "اتصال قطع شد؛ ویندوز خودکار دوباره تلاش می‌کند.": "Disconnected. Windows will retry automatically.",
            "کد اتصال از ویندوز دریافت نشد؛ دوباره تلاش کن.": "No pairing code received. Try again.",
            "کد اتصال اشتباه است.": "Incorrect pairing code.",
            "کابل یا برنامه ویندوز قطع شد.": "Cable or Windows app disconnected.",
            "USB متوقف شد.": "USB stopped.",
            "ارسال عقب افتاده؛ متوقف شد. دوباره شروع کن.": "Input queue was delayed. Enable pen again.",
            "اشتراک تصویر متوقف شد؛ برنامه ویندوز را دوباره اجرا کن.": "Display sharing stopped. Check the Windows app.",
            "پیام نامعتبر است.": "Invalid message.",
            "پیام ویندوز قابل خواندن نیست.": "Unable to read the Windows message.",
            "پیام ناشناخته از ویندوز.": "Unknown Windows message.",
            "نسخه ویندوز هم باید v0.0.9 باشد.": "Use the matching Windows version v0.0.9.",
            "تنظیمات": "Settings",
            "آمادهٔ نوشتن": "Ready to write",
            "از قلم تا کامپیوتر": "From pencil to PC",
            "آیپد و کامپیوتر را با کابل وصل کن.": "Connect your iPad and PC with a cable.",
            "کد اتصال": "Pairing code",
            "این کد را در LearkPeN ویندوز وارد کن.": "Enter this code in LearkPeN on Windows.",
            "آمادگی آیپد": "iPad status",
            "آماده": "Ready",
            "شروع اتصال را بزن": "Tap Start connection",
            "اتصال به کامپیوتر": "PC connection",
            "متصل": "Connected",
            "در انتظار": "Waiting",
            "تصویر زنده": "Live display",
            "در حال دریافت": "Receiving",
            "در انتظار تصویر": "Waiting for video",
            "توقف اتصال": "Stop connection",
            "شروع اتصال": "Start connection",
            "نمایش تصویر": "Show display",
            "اتصال": "Connection",
            "توقف قلم": "Pause pen",
            "شروع قلم": "Enable pen",
            "نمای کامل": "Fit display",
            "ظاهر LearkPeN": "LearkPeN appearance",
            "ظاهر": "Appearance",
            "سیستم": "System",
            "روشن": "Light",
            "تیره": "Dark",
            "صاف کردن دست‌خط": "Handwriting stabilizer",
            "سریع": "Fast",
            "متعادل": "Balanced",
            "نرم‌تر": "Smoother",
            "نرمی بیشتر، خط صاف‌تر و کمی تأخیر بیشتر.": "More smoothing gives steadier lines with slightly more delay.",
            "حساسیت فشار": "Pressure sensitivity",
            "فشار در حالت موس، ضخامت قلم ویندوز را تغییر نمی‌دهد.": "Pressure does not change Windows brush width in mouse mode.",
            "تمام": "Done",
            "کنترل‌ها": "Controls",
            "زبان": "Language"
        ]
        return translations[text] ?? text
    }
    private func revealToolbar() {
        toolbarHide?.cancel()
        withAnimation(reduceMotion ? nil : .easeInOut(duration: 0.2)) { toolbarVisible = true }
        let hide = DispatchWorkItem {
            guard writing, !showSettings else { return }
            withAnimation(reduceMotion ? nil : .easeInOut(duration: 0.25)) { toolbarVisible = false }
        }
        toolbarHide = hide
        DispatchQueue.main.asyncAfter(deadline: .now() + 4, execute: hide)
    }
    private var splash: some View {
        ZStack {
            background.ignoresSafeArea()
            Circle().fill(learkSky.opacity(dark ? 0.12 : 0.07))
                .frame(width: 260, height: 260).blur(radius: 55)
                .scaleEffect(reduceMotion ? 1 : (splashReveal ? 1.1 : 0.65))
                .opacity(splashReveal ? 1 : 0)
            VStack(spacing: 20) {
                Image(systemName: "pencil.tip")
                    .font(.system(size: 30, weight: .light)).foregroundStyle(learkSky)
                    .offset(y: reduceMotion ? 0 : (splashReveal ? 0 : 10))
                    .opacity(splashReveal ? 1 : 0)
                Text("learkcompany")
                    .font(.system(size: 36, weight: .light, design: .rounded))
                    .tracking(reduceMotion ? 2 : (splashReveal ? 2 : 7))
                    .foregroundStyle(learkSky)
                    .blur(radius: reduceMotion || splashReveal ? 0 : 5)
                    .opacity(splashReveal ? 1 : 0)
                    .scaleEffect(reduceMotion ? 1 : (splashReveal ? 1 : 0.96))
                Capsule().fill(LinearGradient(colors: [learkSky.opacity(0), learkSky, learkSky.opacity(0)],
                                               startPoint: .leading, endPoint: .trailing))
                    .frame(width: 160, height: 2)
                    .scaleEffect(x: reduceMotion ? 1 : (splashAccent ? 1 : 0.05), y: 1)
                    .opacity(splashAccent ? 1 : 0)
            }.padding(30)
        }
        .opacity(splashExit ? 0 : 1)
        .scaleEffect(reduceMotion ? 1 : (splashExit ? 1.025 : 1))
        .accessibilityElement(children: .ignore).accessibilityLabel("learkcompany")
        .zIndex(10)
    }
    private var languageWelcome: some View {
        ZStack {
            background.ignoresSafeArea()
            VStack(spacing: 28) {
                Text("LearkPeN").font(.largeTitle.bold()).foregroundStyle(learkSky)
                Text("Choose your language / زبان را انتخاب کن").multilineTextAlignment(.center)
                Picker("Language", selection: $language) {
                    Text("English").tag("en")
                    Text("فارسی").tag("fa")
                }.pickerStyle(.segmented)
                LearkButton(title: language == "fa" ? "ادامه" : "Continue", icon: "arrow.right", prominent: true) { languageChosen = true }
            }.padding(32).frame(maxWidth: 480)
        }.zIndex(9)
    }
    private var connectionPage: some View {
        ScrollView {
            VStack(spacing: 22) {
                HStack {
                    Text("LearkPeN").font(.title2.weight(.bold)).foregroundStyle(learkSky)
                    Spacer()
                    LearkButton(title: tr("تنظیمات"), icon: "slider.horizontal.3") { showSettings = true }
                }.padding(.bottom, 12)
                LearkConnectionMark(waiting: model.listening && !model.connected, connected: model.connected)
                VStack(spacing: 8) {
                    Text(model.connected ? tr("آمادهٔ نوشتن") : tr("از قلم تا کامپیوتر")).font(.largeTitle.weight(.bold))
                    Text(tr("آیپد و کامپیوتر را با کابل وصل کن.")).foregroundStyle(.secondary)
                }
                VStack(spacing: 8) {
                    Text(tr("کد اتصال")).font(.subheadline).foregroundStyle(.secondary)
                    Text(model.pairingCode).font(.system(size: 76, weight: .light, design: .rounded))
                        .monospacedDigit().tracking(12).foregroundStyle(learkSky)
                    Text(tr("این کد را در LearkPeN ویندوز وارد کن.")).font(.caption).foregroundStyle(.secondary)
                }.frame(maxWidth: .infinity).padding(24)
                    .background(learkSky.opacity(dark ? 0.12 : 0.06), in: RoundedRectangle(cornerRadius: 28))
                VStack(spacing: 16) {
                    statusRow(tr("آمادگی آیپد"), detail: model.listening ? tr("آماده") : tr("شروع اتصال را بزن"), ready: model.listening)
                    statusRow(tr("اتصال به کامپیوتر"), detail: model.connected ? tr("متصل") : tr("در انتظار"), ready: model.connected)
                    statusRow(tr("تصویر زنده"), detail: model.videoReady && model.fps > 0 ? tr("در حال دریافت") : tr("در انتظار تصویر"), ready: model.videoReady && model.fps > 0)
                }.padding(22).background(Color.primary.opacity(0.035), in: RoundedRectangle(cornerRadius: 24))
                Text(tr(model.status)).font(.footnote).foregroundStyle(.secondary).multilineTextAlignment(.center)
                    .frame(maxWidth: .infinity)
                HStack(spacing: 12) {
                    LearkButton(title: model.listening ? tr("توقف اتصال") : tr("شروع اتصال"), icon: model.listening ? "stop.fill" : "cable.connector", prominent: true) {
                        if model.listening { model.stopUSB() } else { model.startUSB() }
                    }
                    LearkButton(title: tr("نمایش تصویر"), icon: "display") { writing = true; revealToolbar() }.disabled(!model.connected)
                }
                Text("LearkPeN · v0.0.9").font(.caption2).foregroundStyle(.tertiary).padding(.top, 8)
            }.frame(maxWidth: 540).padding(28).frame(maxWidth: .infinity)
        }
    }
    private func statusRow(_ title: String, detail: String, ready: Bool) -> some View {
        HStack(spacing: 12) {
            Image(systemName: ready ? "checkmark.circle.fill" : "circle.dotted")
                .foregroundStyle(ready ? learkSky : Color.secondary)
            Text(title).font(.subheadline.weight(.medium))
            Spacer()
            Text(detail).font(.caption).foregroundStyle(.secondary)
        }.animation(reduceMotion ? nil : .easeInOut(duration: 0.25), value: ready)
    }
    private var desktop: some View {
        ZStack(alignment: .top) {
            GeometryReader { area in
                PencilPad(model: model).aspectRatio(model.screenAspect, contentMode: .fit)
                    .frame(width: area.size.width, height: area.size.height)
            }.ignoresSafeArea()
            if toolbarVisible {
            ScrollView(.horizontal, showsIndicators: false) {
                HStack(spacing: 10) {
                    LearkButton(title: tr("اتصال"), icon: "cable.connector") { model.pauseWriting(); writing = false }
                    Text("\(model.fps) / 60 FPS").font(.caption.monospacedDigit()).padding(.horizontal, 8)
                    LearkButton(title: model.enabled ? tr("توقف قلم") : tr("شروع قلم"), icon: "pencil.tip", prominent: true) {
                        model.togglePen()
                    }.disabled(!model.connected || !model.videoReady || model.fps == 0)
                    LearkButton(title: tr("نمای کامل"), icon: "arrow.up.left.and.arrow.down.right") { model.resetViewID += 1 }
                    LearkButton(title: tr("تنظیمات"), icon: "slider.horizontal.3") { model.suspendInput(); showSettings = true }
                }.padding(8)
            }.background(.ultraThinMaterial, in: RoundedRectangle(cornerRadius: 24)).padding(.horizontal, 12).padding(.top, 6)
                .simultaneousGesture(TapGesture().onEnded { revealToolbar() })
                .simultaneousGesture(DragGesture(minimumDistance: 0).onChanged { _ in revealToolbar() })
            } else {
                HStack {
                    LearkButton(title: tr("کنترل‌ها"), icon: "ellipsis") { revealToolbar() }
                    Spacer()
                }.padding(12)
            }
        }
    }
    private var settings: some View {
        NavigationStack {
            ScrollView {
                VStack(alignment: .leading, spacing: 24) {
                    Text(tr("زبان"))
                    Picker(tr("زبان"), selection: $language) {
                        Text("English").tag("en")
                        Text("فارسی").tag("fa")
                    }.pickerStyle(.segmented)
                    Text(tr("ظاهر LearkPeN")).font(.headline)
                    Picker(tr("ظاهر"), selection: $appearance) {
                        Text(tr("سیستم")).tag("system")
                        Text(tr("روشن")).tag("light")
                        Text(tr("تیره")).tag("dark")
                    }.pickerStyle(.segmented)
                    Divider()
                    Text(tr("صاف کردن دست‌خط")).font(.headline)
                    HStack {
                        LearkButton(title: tr("سریع"), icon: "bolt") { model.smoothing = 0.35 }
                        LearkButton(title: tr("متعادل"), icon: "pencil") { model.smoothing = 0.75 }
                        LearkButton(title: tr("نرم‌تر"), icon: "water.waves") { model.smoothing = 0.95 }
                    }
                    Slider(value: $model.smoothing, in: 0...1)
                    Text(tr("نرمی بیشتر، خط صاف‌تر و کمی تأخیر بیشتر.")).font(.caption).foregroundStyle(.secondary)
                    Text(tr("حساسیت فشار")).font(.headline)
                    Slider(value: $model.pressureSensitivity, in: 0.5...2)
                    Text(tr("فشار در حالت موس، ضخامت قلم ویندوز را تغییر نمی‌دهد.")).font(.caption).foregroundStyle(.secondary)
                    LearkButton(title: tr("تمام"), icon: "checkmark", prominent: true) { showSettings = false }
                }.padding(28).frame(maxWidth: 580).frame(maxWidth: .infinity)
            }.background(background).navigationTitle(tr("تنظیمات")).navigationBarTitleDisplayMode(.inline)
        }.tint(learkSky).preferredColorScheme(selectedScheme)
    }
}

@main
struct LearkPeNApp: App {
    var body: some Scene { WindowGroup { ContentView() } }
}
