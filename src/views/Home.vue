<template>
  <div>
    <div style="position:absolute;top:0px;left:10px;width:900px;height:800px;z-index:0" ref="mazeControl">
    </div>
    <div style="position:absolute;top:0px;left:10px;width:900px;height:800px;z-index:100" ref="arrowControl">
    </div>
  </div>
</template>

<script>
import Matter from 'matter-js'

export default {
  name: 'home',
  data () {
    return {
      maze: null,
      arrow: null,
      rock: null,
      elastic: null,
      ball: null,
      lastpos: { x: 25, y: 750 },
      isDragging: false
    }
  },
  mounted () {
    this.maze = this.initMaze()
    this.createBall(this.lastpos)
    let that = this
    setInterval(function () {
      if (that.lastpos.x !== Math.round(that.ball.position.x) || that.lastpos.y !== Math.round(that.ball.position.y)) {
        that.removeArrow()
        that.lastpos = { x: Math.round(that.ball.position.x), y: Math.round(that.ball.position.y) }
      } else {
        if (!(!!that.rock && !!that.elastic)) {
          that.createNewArrow(that.lastpos)
        }
      }
    }, 500)
  },
  methods: {
    initMaze () {
      let that = this
      let Engine = Matter.Engine
      let Render = Matter.Render
      let Runner = Matter.Runner
      // let Composites = Matter.Composites
      let Events = Matter.Events
      // let Constraint = Matter.Constraint
      let MouseConstraint = Matter.MouseConstraint
      let Mouse = Matter.Mouse
      let World = Matter.World
      let Bodies = Matter.Bodies

      // create engine
      let engine = Engine.create()
      // let world = engine.world

      // create renderer
      let render = Render.create({
        element: that.$refs.mazeControl,
        engine: engine,
        options: {
          width: 900,
          height: 800,
          background: '#white',
          wireframes: false
        }
      })
      Render.run(render)

      // create arrow renderer
      let engineA = Engine.create()
      // let worldA = engineA.world

      let renderA = Render.create({
        element: that.$refs.arrowControl,
        engine: engineA,
        options: {
          width: 900,
          height: 800,
          background: 'transparent',
          wireframes: false
        }
      })

      Render.run(renderA)

      // create runner
      let runner = Runner.create()
      Runner.run(runner, engine)

      let runnerA = Runner.create()
      Runner.run(runnerA, engineA)

      // add bodies
      let framewalls = []
      framewalls.push(Bodies.rectangle(
        450,
        5,
        900,
        10, { density: 1, isStatic: true }))
      framewalls.push(Bodies.rectangle(
        450,
        795,
        900,
        10, { density: 1, isStatic: true }))
      framewalls.push(Bodies.rectangle(
        5,
        400,
        10,
        800, { density: 1, isStatic: true }))
      framewalls.push(Bodies.rectangle(
        895,
        400,
        10,
        800, { density: 1, isStatic: true }))
      framewalls.push(Bodies.rectangle(
        40,
        0 + 370,
        60,
        740, { density: 1, isStatic: true }))
      framewalls.push(Bodies.rectangle(
        860,
        70 + 370,
        60,
        740, { density: 1, isStatic: true }))

      let walls = []
      // for (let i = 0; i < 4; i++) {
      //   for (let j = 0; j < 4; j++) {
      //     let n = (i * 4 + j) % 7
      //     if (n < 3) {
      //       walls.push(Bodies.rectangle(
      //         150 + j * 200,
      //         100 + i * 200,
      //         200,
      //         10, { density: 1, isStatic: true }))
      //     } else {
      //       walls.push(Bodies.polygon(
      //         150 + j * 200,
      //         100 + i * 200,
      //         n, 100, { density: 1, isStatic: true }))
      //     }
      //   }
      // }
      for (let i = 1; i < 800; i++) {
        framewalls.push(Bodies.rectangle(
          70 + (830 - 70 - 60 * (i % 2 === 0 ? 1 : -1)) / 2,
          10 + 5 + 60 * i,
          830 - 70 - 60,
          10, { density: 1, isStatic: true }))
      }

      engine.world.gravity.x = 0
      engine.world.gravity.y = 0
      engine.world.gravity.scale = 0.01
      engineA.world.gravity.x = 0
      engineA.world.gravity.y = 0
      engineA.world.gravity.scale = 0.01
      World.add(engine.world, framewalls)
      World.add(engine.world, walls)
      // World.add(engine.world, [rock, elastic])

      // add mouse control
      let mouse = Mouse.create(renderA.canvas)
      let mouseConstraint = MouseConstraint.create(engineA, {
        mouse: mouse,
        constraint: {
          stiffness: 0.2,
          render: {
            visible: false
          }
        }
      })

      World.add(engineA.world, mouseConstraint)
      Events.on(mouseConstraint, 'mousemove', function () {
        if (that.isDragging) {
          let mousedown = mouseConstraint.mouse.mousedownPosition
          let mousepos = mouseConstraint.mouse.position
          let arrowAngle = Math.atan(Math.abs(mousepos.y - mousedown.y) / Math.abs(mousepos.x - mousedown.x)) *
                ((mousepos.y - mousedown.y) / Math.abs(mousepos.y - mousedown.y)) *
                ((mousepos.x - mousedown.x) / Math.abs(mousepos.x - mousedown.x)) +
                Math.PI * (mousepos.x - mousedown.x > 0 ? 0 : 1)
          if (!isNaN(arrowAngle) && that.rock) {
            Matter.Body.setAngle(that.rock, arrowAngle)
          }
        }
      })
      Matter.Events.on(mouseConstraint, 'startdrag', function () {
        that.isDragging = true
      })

      Matter.Events.on(mouseConstraint, 'enddrag', function () {
        let x = (that.rock.position.x - that.elastic.pointA.x) / 10
        let y = (that.rock.position.y - that.elastic.pointA.y) / 10
        if (!(isNaN(x) || isNaN(y))) {
          that.isDragging = false
          that.removeArrow()
          Matter.Body.setVelocity(that.ball, { x: x, y: y })
        }
      })

      Events.on(engine, 'afterUpdate', function () {
        // if (mouseConstraint.mouse.button === -1 && (rock.position.x > 190 || rock.position.y < 430)) {
        //   rock = Bodies.polygon(170, 450, 7, 20, rockOptions)
        //   World.add(engine.world, rock)
        //   elastic.bodyB = rock
        // }

        for (let i = 0; i < walls.length; i++) {
          Matter.Body.rotate(walls[i], r[(step + i) % r.length] * Math.PI / walls.length * n[(step + i) % n.length])
        }
      })

      let r = [1, 0.5, -0.5, -1]
      let n = [1, 2, 3, 1, 1.5]
      let step = 0

      // keep the mouse in sync with rendering
      renderA.mouse = mouse

      // fit the render viewport to the scene
      Render.lookAt(render, {
        min: { x: 0, y: 0 },
        max: { x: 900, y: 800 }
      })
      Render.lookAt(renderA, {
        min: { x: 0, y: 0 },
        max: { x: 900, y: 800 }
      })

      // context for MatterTools.Demo
      return {
        engineA: engineA,
        runnerA: runnerA,
        renderA: renderA,
        canvasA: renderA.canvas,
        engine: engine,
        runner: runner,
        render: render,
        canvas: render.canvas,
        stop: function () {
          Matter.Render.stop(render)
          Matter.Runner.stop(runner)
          Matter.Render.stop(renderA)
          Matter.Runner.stop(runnerA)
        }
      }
    },
    createBall (obj) {
      let x = obj.x
      let y = obj.y
      this.ball = Matter.Bodies.polygon(x, y, 20, 20, {
        density: 1,
        render: {
          sprite: {
            texture: './img/ball.png'
          }
        }
      })
      Matter.World.add(this.maze.engine.world, [this.ball])
    },
    createNewArrow (obj) {
      // this.removeArrow()
      let x = obj.x
      let y = obj.y
      this.rock = Matter.Bodies.polygon(x, y, 20, 20, {
        density: 1,
        render: {
          sprite: {
            texture: './img/redarrow.png'
          }
        }
      })

      this.elastic = Matter.Constraint.create({
        pointA: { x: x, y: y },
        bodyB: this.rock,
        stiffness: 0.05,
        render: {
          visible: true,
          lineWidth: 10,
          strokeStyle: '#ff0000',
          type: 'line',
          anchors: true
        }
      })
      Matter.World.add(this.maze.engineA.world, [this.rock, this.elastic])
    },
    removeArrow: function () {
      if (!!this.rock && !!this.elastic) {
        Matter.World.remove(this.maze.engineA.world, this.rock)
        Matter.World.remove(this.maze.engineA.world, this.elastic)
        this.rock = null
        this.elastic = null
      }
    }
  }
}
</script>
