---
layout: post
title: "Tartu: an iOS game in Swift/SpriteKit"
categories: [ios, gamedev, swiftlang]
---

It's now time to clean up some of my old projects and publish this little iOS game I made last year.
I had two main objective: 1) Make some money and 2) Learn the basic of _Swift_.

Here's a description of the main parts and some explanation how to glue things together when developing a mobile game of iOS.


![tartu1](/images/tartu/tartu2.png)
<small> Tartu is getting ready to start. Candies... </small>


## Why learning Swift

It's well explained by [Martin Fowler](http://martinfowler.com/bliki/OneLanguage.html) on his blog:

> For many developers, the one-language notion is a sign of lack of professionalism.
This is best exemplified by the Pragmatic Programmers' advice to learn a new language every year.
The point here is that **programming languages do affect the way you think about programming**,
and learning new languages can do a lot to help you think about solving problems in different ways.
(It's important to learn languages that are quite different in order to get the benefit of this. Java and C# are too similar to count.)


Since I knew nothing of iOS programming I thought that making a game would be more fun that programming yet another calculator. This online
[iOS Swift course](https://www.youtube.com/watch?v=_lRx1zoriPo) from  Stanford University is a *must* (apart from the calculator).

Obviously, I also wanted to get rich as game developer, and I was quite surprised I even made money with it.
To be precise I made $0.01 (and counting...) in 1 year.

The game mechanics is simple: there's a _turtle_ riding a unycicle collecting candys and trying to avoid spikes.
It's a typical one tap scroller, but complex enough to learn several aspect of iOS/Swift programming.


## Code overview

Here's a brief description of the things I've learnt during the development of this iOS game.
I'd recommed to follow some good gamedev videos here: [CartoonSmart](http://cartoonsmart.com/ios-and-tvos-video-tutorials/).
(Note: I am not affiliated and I genuinely think they have good videos).


### Optional Type in Swift

Before looking at the Game, a quick comment on Swift syntax. Some types have a postfix ? char. That's just
syntactic sugar for [Optionals Type](https://developer.apple.com/library/content/documentation/Swift/Conceptual/Swift_Programming_Language/Types.html#//apple_ref/doc/uid/TP40014097-CH31-ID452).

An quick example is :

{% highlight swift %}
    var str:String? = "hello"

    if let a = str {
        print(str)
    } else {
        print("not defined!")
    }
{% endhighlight %}


In Scala, we could do something similar with:

{% highlight scala %}
    val str:Option[String] = Some("hello")

    str match {
        case Some(value) => print(value)
        case _ => print("not defined")
    }
{% endhighlight %}

### The GameViewController

The _GameViewController_ is the class the initialise the game:


{% highlight swift %}
class GameViewController: UIViewController {

    override func viewDidLoad() {
        super.viewDidLoad()
        // Configure the view.
        let skView = self.view as! SKView

        skView.showsFPS = true
        skView.showsNodeCount = true
        if #available(iOS 8.0, *) {
            skView.showsPhysics = true
        } else {
            // Fallback on earlier versions
        }

        /* Sprite Kit applies additional optimizations to improve rendering performance */
        skView.ignoresSiblingOrder = true

        let scene = GameScene()
        scene.scaleMode = SKSceneScaleMode.ResizeFill
        scene.size = skView.bounds.size

        if scene.size.width < 480 {
            scene.size.width = 480

        } else if scene.size.height < 320 {
            scene.size.height = 320
        }
        skView.presentScene(scene)
    }
{% endhighlight %}

The _GameScene_ is loaded and the presented to the screen. While debugging is useful to display _FPS_ and physics borders
(for detecting problems with sprite collisions).


The GameScene loads the background, the sprites, and background layers. The game state is saved in the global
variable _currentGameState_, and it represents the possible states of the game (_Active_, _Death_, _Completed_).



{% highlight swift %}
class GameScene: SKScene, SKPhysicsContactDelegate {

    var currentLevel = 1
    let layerBackgroundSlow = LayerBackground()
    let layerBackgroundFast = LayerBackground()
    let layerGameWorld = LayerWorld(
        tileSize: CGSize(width: 20, height: 20),
        gridSize: CGSize(width: 200, height: 15))

    let player = Player(size: CGSize(width: 30 , height: 45))
    var currentGameState = GameState.Intro
    var lastUpdateTime:NSTimeInterval = 0
    var dt:NSTimeInterval = 0
    var playerStartPoint:CGPoint?

    override func didMoveToView(view: SKView) {
        physicsWorld.contactDelegate = self
        physicsWorld.gravity = CGVector(dx: 0, dy: -7)
        anchorPoint = CGPoint(x: 0.5, y: 0.5)

        if let level = settings.levelData(currentLevel) {
            addSun()
            addSea(images: level.sea)
            addSky(color: level.skyColor)
            addBackground(parallax: level.parallax)
            addGameWorld(tmxfile: level.tmxfile)

            if currentGameState == .Intro {
                displayTapToStart()
            } else {
                addPlayer()
            }
        }
    }
{% endhighlight %}



The Game loop is the main procedure that continuosly run and _update_ the game status. It's also responsible of listening
to events and firing the correspondent actions. When the game is _Active_, the player position and the backgrounds layer
are updated. We also check if the player collided with a _spike_ or if the player reached the end of the level.


{% highlight swift %}
    override func update(currentTime: CFTimeInterval) {
        // Delta Time
        if lastUpdateTime > 0 {
            dt = currentTime - lastUpdateTime
        } else {
            dt = 0
        }
        lastUpdateTime = currentTime

        //Update Game
        switch currentGameState {
        case .Active:
            layerBackgroundSlow.update(dt, affectAllNodes: true, parallax: true)
            layerBackgroundFast.update(dt, affectAllNodes: true, parallax: true)
            player.update(dt)

            if player.spikeTouched {
                currentGameState = .Death
            } else if player.completed {
                currentGameState = .Completed
            }

        case .Death:
            player.deathAnimation()
            displayPostScreen()
            currentGameState = .Paused
        case .Completed:
            player.happyAnimation()
            displayNextScreen()
            currentGameState = .Paused
        default:
            break
        }
    }
{% endhighlight %}


### Player and Collisions

Collisions represent how the player interacts with the scene. For example touching a spike  or collecting a candy.

The player is represented with a class, and the collision part is easily managed by SpriteKit Physics:

{% highlight swift %}
    func collidedWith(body:SKPhysicsBody, contact:SKPhysicsContact) {
        if body.categoryBitMask == ColliderType.Wall {
            currentState = State.OnGround
            playGroundSound()
        } else if body.categoryBitMask == ColliderType.Spike {
            spikeTouched = true
            playSpikeSound()
        } else if body.categoryBitMask == ColliderType.Edge {
            spikeTouched = true
            playWaterSound()
        } else if body.categoryBitMask == ColliderType.Trigger {
            completed = true
        }
    }
{% endhighlight %}

In this example, two bodies collide and we use bitmask to check the possibile combinations.


### Sequence of Animations in SpriteKit

While the player is happily jumping and avoiding spikes, there are few animations in the background.
The sun has a smooth shine effect, and two backgrounds are scrolling at different speed with a simple parallax effect.

![tartu1](/images/tartu/tartu1.png)
<small> Sun is shining! The Sun animation is made of four animations running in sequence</small>


Animations are quite easy to implement in SpriteKit, here's an example of how to _draw_ the shining sun:

{% highlight swift %}
    func runAnimation(){

        let scaleUp = SKAction.scaleBy(1.2, duration: 1)
        let scaleDown = SKAction.reversedAction(scaleUp)()
        let fadeOut = SKAction.fadeOutWithDuration(1)
        let fadeIn = SKAction.reversedAction(fadeOut)()

        let innerBorder = circleOfRadius(size.width/2)
        innerBorder.strokeColor = SKColor.whiteColor()
        innerBorder.lineWidth = 5
        addChild(innerBorder)

        let outBorder = circleOfRadius(size.width/2 + size.width*0.1)
        outBorder.fillColor = SKColor.whiteColor()
        outBorder.alpha = 0.2
        addChild(outBorder)

        let outBorderAnim = SKAction.sequence([scaleUp, fadeOut, scaleDown])

        let explosion = SKAction.runBlock({
            outBorder.runAction(outBorderAnim)
            outBorder.alpha = 0.2
        })
        let wait = SKAction.waitForDuration(5)

        let innerBorderAnim = SKAction.sequence([
            scaleUp,
            SKAction.group([fadeOut, explosion,wait]),
            scaleDown,
            fadeIn
            ])

        innerBorder.runAction(SKAction.repeatActionForever(innerBorderAnim))
    }
{% endhighlight %}

There are four animations: to scale the sun _Up_ and _Down_ and to fade the shine _In_ and _Out_.
The animation is made of two sequences, the _outBorderAnim_ and the _innerBorderAnim_.
Chaining of sequences is donw with `SKAction.sequence`, and if we want to repeat the sequence `SKAction.repeatActionForever`.



### Levels construction

Levels are: (drumrolls) just _XML_. A Tilemap editor is used to position each tile. I used [Tiled](http://www.mapeditor.org/) for that.
After a level is designed it's necessary to parse the XML. I was using [JSTileMap](https://github.com/slycrel/JSTileMap) to read the XML
(although it's in Objective-C), and ad-hoc Swift code to create the Sprites.

{% highlight swift %}
class LayerWorld:LayerTiles {

    var tileMap:JSTileMap? = nil

    func addTileMap(filename:String) {
        tileMap = JSTileMap(named: filename)
        if tileMap != nil {

            tileMap!.name = "tileMap"
            let mapBounds = tileMap!.calculateAccumulatedFrame()

            addGroup("walls", tilemap: tileMap!) { wall in
                if let point = self.objectPoint(wall), size = self.objectSize(wall){
                    self.tileMap!.addChild(Wall(offset: point, size: size))
                }
            }

            addGroup("spikes", tilemap: tileMap!) { spike in
                if let point = self.objectPoint(spike), size = self.objectSize(spike){
                    self.tileMap!.addChild(Spike(offset: point, size: size))
                }
            }

            addGroup("triggers", tilemap:  tileMap!) {trigger in
                if let point = self.objectPoint(trigger), polyline = self.objectPolyline(trigger){
                        if let name = self.objectName(trigger) {
                            if name == "completed" {
                                self.tileMap!.addChild(
                                    Trigger(offset: point, polyline: polyline, name:name))
                            }
                        } else {
                            self.tileMap!.addChild(Edge(offset: point, polyline: polyline))
                        }
                }
            }
            addChild(tileMap!)
        }
    }
}
{% endhighlight %}


![tartu1](/images/tartu/tartu3.png)
<small>  Winter is coming </small>

## Conclusion

Swift is a fine language with interesting features. I guess I still need to build a calculator though and follow the Stanford Course.
Many thanks to [Neil North](https://www.udemy.com/user/neilnorth/) for his online course. Some of the code of Tartu is taken
from his excellent tutorials.

The entire code of the game is published on my [github](https://github.com/rosario/Tartu). Sounds effect were freely downloaded from... somewhere, I can't remember
(if you recognise your sound get in touch and I'll add a link here). Graphics, background and tiles were made by myself,
so feel free to use them.

I've been careful to place any copyright section when required, but if I've forgot please get in touch and I'll fix it.




