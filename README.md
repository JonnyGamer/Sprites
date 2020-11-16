# Sprites

A nicely built SpriteKit framework.

# Scene Setup

```swift
struct Scene1: Hostable {
    init() { this = SKNode(); that = SKNode() }
    
    var that: SKNode
    var this: SKNode
    
    func begin() {}
    func update() {}
    mutating func touchBegan(_ pos: CGPoint, _ nodes: [SKNode]) {}
    mutating func touchMoved(_ moved: CGVector) {}
    mutating func touchEnded(_ pos: CGPoint, _ nodes: [SKNode]) {}
    mutating func reset() {}
    
}
```

# Framework Code

```swift

import UIKit
import SpriteKit
import GameplayKit

@UIApplicationMain
class AppDelegate: UIResponder, UIApplicationDelegate {
    var window: UIWindow?
}

var mega: CGFloat = 1
var w: CGFloat = 1000
let h: CGFloat = 1000
var scene = Scene()
var fonty: CGFloat = 20.0 // 9.0

class GameViewController: UIViewController {

    override func viewDidLoad() {
        super.viewDidLoad()
        
        if let view = self.view as? SKView {
            w = (UIScreen.main.bounds.size.width / UIScreen.main.bounds.size.height) * 1000
            scene.scaleMode = .aspectFit
            view.presentScene(scene)
            view.ignoresSiblingOrder = true
            view.showsFPS = true
            view.showsNodeCount = true
        }
    }

    override var shouldAutorotate: Bool {
        return true
    }

    override var supportedInterfaceOrientations: UIInterfaceOrientationMask {
        if UIDevice.current.userInterfaceIdiom == .phone {
            return .allButUpsideDown
        } else {
            return .all
        }
    }

    override var prefersStatusBarHidden: Bool {
        return true
    }
}



import SpriteKit
extension SKNode {
    @discardableResult
    func edit<T: SKNode>(_ this: (T) -> ()) -> T {
        this(self as! T)
        return self as! T
    }
    
    func node<T: SKNode>(_ this: T) -> T {
        addChild(this); return this
    }
    func label(_ w: SKLabelNode) -> SKLabelNode {
        return node(w)
    }
}

extension SKNode: Doable {
    @discardableResult
    func doChildren(_ this: (SKNode) -> ()) -> Self { for i in self.children { this(i) }; return self }
}
protocol Doable { init() }
extension Doable {
    @discardableResult
    func `do`(_ this: (Self) -> ()) -> Self { this(self); return self }
    static func `do`(_ this: (Self) -> ()) -> Self { let foo = self.init(); this(foo); return foo }
}

extension Array where Element: SKNode {
    func `do`(_ this: (Element) -> ()) { for po in self { this(po) } }
    subscript(_ this: String, _ list: String...) -> SKNode? {
        var foo: SKNode? = self.first { $0.name == this }
        for chill in list {
            foo = foo?.children.first { $0.name == chill }
        }
        if foo == nil { print("Warning! self\(list) was not found") }
        return foo
    }
    subscript<T: SKNode>(_ type: T.Type,_ this: String, _ list: String...) -> T? {
        var foo: SKNode? = self.first { $0.name == this }
        for chill in list {
            foo = foo?.children.first { $0.name == chill }
        }
        return foo as? T
    }
}
extension SKNode {
    @discardableResult
    func callAsFunction<T: SKNode>(_ type: T.Type = SKNode.self as! T.Type, _ list: String..., wow: (T) -> ()) -> T? {
        var foo: SKNode? = self
        for chill in list {
            foo = foo?.childNode(withName: chill)
        }
        return foo as? T
    }
    
    func bye() {
        removeFromParent()
    }
}

extension Array where Element == SKLabelNode {
    func accumulatedSize() -> CGSize {
        var bigX: CGFloat = 0
        var bigY: CGFloat = 0
        for i in self {
            bigX = Swift.max(i.frame.width + (i.text?.numberOfLeadingAndTrailingSpaces ?? 0), bigX)
            bigY += i.frame.height + 10
        }
        return .init(width: bigX, height: bigY - 10)
    }
}

var textures = [String:SKTexture]()
func Texture(_ name: String) -> SKTexture {
    if textures[name] == nil {
        textures[name] = SKTexture(imageNamed: name)
    }
    return textures[name]!
}

public func eraseAllData() {
    UserDefaults.standard.removePersistentDomain(forName: Bundle.main.bundleIdentifier!)
    UserDefaults.standard.synchronize()
}

extension Array where Element == SKNode {
    func touchBegan() {
        _ = map {
            ($0 as? SuperTouchable)?.touchBegan()
        }
    }
    func touchEnd() {
        _ = map {
            ($0 as? SuperTouchable)?.touchEnd()
        }
    }
}
extension SKNode {
    var width: CGFloat { frame.width }
    var height: CGFloat { frame.height }
    var halfWidth: CGFloat { return frame.width.half }
    var halfHeight: CGFloat { return frame.height.half }
}

extension CGSize {
    static var fullScreen: CGSize { return .init(width: w, height: h) }
}
extension CGPoint {
    static var midScreen: CGPoint { return .init(x: w / 2, y: h / 2) }
    static var half: CGPoint { return .init(x: 0.5, y: 0.5) }
}
extension CGSize {
    var padding: CGSize {
        return .init(width: width + 20, height: height + 20)
    }
}
extension CGFloat {
    var half: CGFloat { self / 2 }
}

protocol SuperTouchable {
    var touchBegan: () -> () { get set }
    var touchEnd: () -> () { get set }
}


class Scene: SKScene {
    var currentTouches: Set<UITouch> = []
    var host: Hostable = Scene1()
    var previouslyHosted: [String:Hostable] = [:]
    
    override init() {
        super.init(size: .init(width: w, height: h))
        anchorPoint = .zero
        backgroundColor = .white
        
        addChild(host.this)
        addChild(host.that)
        
        host.this.name = "THIS"
        host.that.name = "THAT"
        
        host.begin()
        
        previouslyHosted["\(type(of: host))"] = host
    }
    
    required init?(coder aDecoder: NSCoder) {
        fatalError("init(coder:) has not been implemented")
    }
    
    func detectTouches(_ touches: Set<UITouch>) {
        if checkIfPresenting() { return }
        var nodesFound: Set<SKNode> = []
        for touch in touches {
            currentTouches.insert(touch)
            nodesFound = nodesFound.union(nodes(at: touch.location(in: self)))
        }
    }
    
    func checkIfPresenting() -> Bool { return host.this.hasActions() || host.that.hasActions() }
    
    override func touchesBegan(_ touches: Set<UITouch>, with event: UIEvent?) {
        if checkIfPresenting() { return }
        let thisNodes = touches.reduce([SKNode]()) { (foo, bar) -> [SKNode] in
            if bar.phase == .began {
                return foo + nodes(at: bar.location(in: self))
            } else {
                return foo
            }
        }
        guard let pos = touches.first?.location(in: self) else {return}
        host.touchBegan(pos, thisNodes)
    }
    
    override func touchesEnded(_ touches: Set<UITouch>, with event: UIEvent?) {
        if checkIfPresenting() { return }
        let thisNodes = touches.reduce([SKNode]()) { (foo, bar) -> [SKNode] in
            if bar.phase == .ended || bar.phase == .cancelled {
                return foo + nodes(at: bar.location(in: self))
            } else {
                return foo
            }
        }
        guard let pos = touches.first?.location(in: self) else {return}
        host.touchEnded(pos, thisNodes)
    }
    override func touchesCancelled(_ touches: Set<UITouch>, with event: UIEvent?) {
        if checkIfPresenting() { return }
        touchesEnded(touches, with: event)
    }
    
    override func update(_ currentTime: TimeInterval) {
        if checkIfPresenting() { return }
        host.update()
    }
    
    override func touchesMoved(_ touches: Set<UITouch>, with event: UIEvent?) {
        if checkIfPresenting() { return }
        for i in touches {
            if i.phase == .moved {
                let loc = i.location(in: self)
                let prevLoc = i.previousLocation(in: self)
                let d = CGVector(dx: (loc.x - prevLoc.x), dy: (loc.y - prevLoc.y))
                host.touchMoved(d)
            }
        }
    }
    
}

protocol Hostable {
    var that: SKNode { get set }
    var this: SKNode { get set }
    init()
    func begin()
    func update()
    mutating func touchBegan(_ pos: CGPoint,_ nodes: [SKNode])
    mutating func touchMoved(_ moved: CGVector)
    mutating func touchEnded(_ pos: CGPoint,_ nodes: [SKNode])
    mutating func reset()
}


extension Hostable {
    @discardableResult func Label(_ label: SKLabelNode) -> SKLabelNode { return this.label(label) }
    
    func updateWillEnd() {}
    
    func end(_ with: SKAction) {
        that.run(with) { self.that.removeFromParent() }
        this.run(with) { self.this.removeFromParent(); self.that.removeFromParent() }
    }
    
    func present(_ new: Hostable.Type) {
        
        that.run(.fadeOut(withDuration: 0.2)) {
            self.that.removeFromParent()
        }
        this.run(.fadeOut(withDuration: 0.2)) {
            self.that.removeFromParent()
            self.this.removeFromParent()
            
            if let loaded = scene.previouslyHosted["\(new)"] {
                scene.host = loaded
                scene.host.this.alpha = 0
                scene.host.that.alpha = 0
                if scene.host.this.parent == nil {
                    scene.addChild(scene.host.this)
                }
                if scene.host.that.parent == nil {
                    scene.addChild(scene.host.that)
                }
                scene.host.this.name = "THIS"
                scene.host.that.name = "THAT"
                scene.host.this.run(.fadeIn(withDuration: 0.2))
                scene.host.that.run(.fadeIn(withDuration: 0.2))
                scene.host.reset()
            } else {
                scene.host = new.init()
                scene.host.this.alpha = 0
                scene.host.that.alpha = 0
                scene.host.this.name = "THIS"
                scene.host.that.name = "THAT"
                if scene.host.this.parent == nil {
                    scene.addChild(scene.host.this)
                }
                if scene.host.that.parent == nil {
                    scene.addChild(scene.host.that)
                }
                scene.host.this.run(.fadeIn(withDuration: 0.2))
                scene.host.that.run(.fadeIn(withDuration: 0.2))
                scene.host.begin()
                scene.previouslyHosted["\(new)"] = scene.host
            }
        }
    }
}

extension String {
    var numberOfLeadingSpaces: CGFloat {
        var count: CGFloat = 0
        for i in self {
            if i != " " { return count * fonty }
            count += 1
        }
        return count * fonty
    }
    var numberOfLeadingAndTrailingSpaces: CGFloat {
        var count: CGFloat = 0
        for i in self.reversed() {
            if i != " " { return numberOfLeadingSpaces + (count * fonty) }
            count += 1
        }
        return (count * fonty)
    }
}

```
