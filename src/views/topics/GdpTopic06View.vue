<template>
  <TopicMeta :topic="topic" />
  <div id="Pascal-Brainchon-wrapper">
    <canvas id="Pascal-Brainchon-canvas" width="500" height="500" />
  </div>
</template>

<script lang="ts">
import { defineComponent } from 'vue';
import { indexTopicMap } from '@/data';
import { Topic } from '@/types';
import { fabric } from 'fabric';
import { IEvent } from 'fabric/fabric-impl';
import { calculateLineIntersectInLinearEquation, solveLinearEquation } from '@/utils/geometry';

const topic = indexTopicMap.get(6) as Topic;

/**
 * Type definitions
 */
type Intersection = fabric.Intersection & {
  points?: Array<fabric.Point>
}

type Circle = fabric.Circle & {
  [key: string]: any,
  intersects?: Array<Circle>,
  upLine?: Array<fabric.Line>,
  downLine?: Array<fabric.Line>,
  crossLines?: Array<fabric.Line>,
  label?: Label,
}

type Line = fabric.Line & {
  p1?: Circle,
  p2?: Circle,
  m?: number,
  b?: number,
}

type Coord = {
  x: number,
  y: number,
}

type Label = fabric.Text & {
  // The offset of a label away from the point it attatches to, so that
  // they won't mix in a mess.
  offSet: Coord,
}

/**
 * Utility functions
 */
const log = console.log

const makeLabel = (text: string, offSet={x: 0, y: 0}, fontSize=24): Label => {
  let label = new fabric.Text(text, {
    hasControls: false,
    hasBorders: false,
    evented: false,
    fontSize: fontSize,
  }) as Label
  label.offSet = offSet
  return label
}

function isCircle(point: Circle | {top: number, left: number}): point is Circle {
  return point instanceof fabric.Circle
}

function setLabelToPoint(labels: Array<Label>, points: Array<{top: number, left: number}>): void
function setLabelToPoint(labels: Array<Label>, points: Array<Circle>): void
function setLabelToPoint(labels: Array<Label>, points: Array<Circle|{top: number, left: number}>): void {
  for (let index = 0; index < labels.length; index++) {
    let label = labels[index]
    let point = points[index]
    label.left = point.left! + label.offSet.x
    label.top = point.top! + label.offSet.y
    if (isCircle(point)) {
      point.label = label
    }
  }
}

function makeCircle(point: fabric.Point, radius?: number, fill?: any): Circle
function makeCircle(x: number, y: number, radius?: number, fill?: string): Circle
function makeCircle(x: number | fabric.Point, y?: number, radius?: number, fill?: string): Circle {
  if (x instanceof fabric.Point) {
    y = x.y
    x = x.x
  }
  let point = new fabric.Circle({
    left: x,
    top: y,
    hasControls: false,
    hasBorders: false,
    evented: true,
    radius: radius || 3,
    fill: fill || "black",
    originX: "center",
    originY: "center",
  })
  return point
}

const makeLine = (pt1: Coord | fabric.Point, pt2: Coord | fabric.Point, 
    strokeWidth?: number, stroke?: string): fabric.Line => {
      return new fabric.Line([pt1.x, pt1.y, pt2.x, pt2.y], {
    stroke: stroke || "black",
    hasControls: false,
    hasBorders: false,
    evented: false,
    strokeWidth: strokeWidth || 1,
  })
}

function getIntersectFromLines(line1: fabric.Line, line2: fabric.Line): Coord {
  let l1 = solveLinearEquation({ x: line1.x1, y: line1.y1 }, { x: line1.x2, y: line1.y2 })
  let l2 = solveLinearEquation({ x: line2.x1, y: line2.y1 }, { x: line2.x2, y: line2.y2 })
  return calculateLineIntersectInLinearEquation(l1.m, l1.b, l2.m, l2.b)
}


export default defineComponent(
  {
    setup() {
      return { topic };
    },
    mounted() {
      const canvas = new fabric.Canvas("Pascal-Brainchon-canvas", {
        selection: false,
        backgroundColor: "floralwhite",
      });

      let bottomLine = new fabric.Line(  // # TODO use reduce
        [0, 400, 500, 400],
        {
          stroke: "black",
          hasControls: false,
          hasBorders: false,
          evented: false,
          strokeWidth: 2,
        }
      ) as Line;
      Object.assign(bottomLine, solveLinearEquation(
        { x: bottomLine.x1, y: bottomLine.y1 }, { x: bottomLine.x2, y: bottomLine.y2 }
      ))
      // y = -0.12x + 100
      let upperLine = new fabric.Line(
        [0, 130, 500, 40],
        {
          stroke: "black",
          hasControls: false,
          hasBorders: false,
          evented: false,
          strokeWidth: 2,
        }
      ) as Line;
      Object.assign(upperLine, solveLinearEquation(
        { x: upperLine.x1, y: upperLine.y1 }, { x: upperLine.x2, y: upperLine.y2 }
      ))

      const aLabel = makeLabel("A", {x: -20, y: 15});
      const bLabel = makeLabel("B", {x: 0, y: 15});
      const cLabel = makeLabel("C", {x: 5, y: 15});
      const aprimeLabel = makeLabel("A'", {x: -20, y: -40});
      const bprimeLabel = makeLabel("B'", {x: 0, y: -40});
      const cprimeLabel = makeLabel("C'", {x: 0, y: -40});
      const xLabel = makeLabel("X", {x: 20, y: 0})
      const yLabel = makeLabel("Y", {x: -5, y: 20})
      const zLabel = makeLabel("Z", {x: -30, y: 0})


      const pointA = makeCircle(50, 400)
      const pointB = makeCircle(250, 400)
      const pointC = makeCircle(450, 400)
      const pointPrimeA = makeCircle(50, upperLine.m! * 50 + upperLine.b!)
      const pointPrimeB = makeCircle(250, upperLine.m! * 250 + upperLine.b!)
      const pointPrimeC = makeCircle(450, upperLine.m! * 450 + upperLine.b!)

      const movablePoints = [pointA, pointB, pointC, pointPrimeA, pointPrimeB, pointPrimeC]

      setLabelToPoint(
        [aLabel, bLabel, cLabel, aprimeLabel, bprimeLabel, cprimeLabel],
        movablePoints
      )

      let pointsFromMovables = movablePoints.map(ele => new fabric.Point(ele.left!, ele.top!))
      const [pA, pB, pC, ppA, ppB, ppC] = pointsFromMovables
      const aBprime = makeLine(pA, ppB, undefined, "blue")
      const aCprime = makeLine(pA, ppC, undefined, "blue")
      const bAprime = makeLine(pB, ppA, undefined, "blue")
      const bCprime = makeLine(pB, ppC, undefined, "blue")
      const cAprime = makeLine(pC, ppA, undefined, "blue")
      const cBprime = makeLine(pC, ppB, undefined, "blue")

      // aBprime.set({x1: 60})
      const pZ = (fabric.Intersection.intersectLineLine(
        pA, ppB, ppA, pB) as Intersection).points![0]
      const pY = (fabric.Intersection.intersectLineLine(
        pA, ppC, ppA, pC) as Intersection).points![0]
      const pX = (fabric.Intersection.intersectLineLine(
        pB, ppC, pC, ppB) as Intersection).points![0]
      const [pointX, pointY, pointZ] = [pX, pY, pZ].map(p => makeCircle(p, undefined, 3))
      log("pointX", pointX)
      setLabelToPoint([zLabel, yLabel, xLabel], [pointZ, pointY, pointX])

      const COLL_OFF_SET = 100
      const collinearLine = makeLine(pZ, pX, 2, "red") as Line
      collinearLine.p1 = pointZ
      collinearLine.p2 = pointX
      Object.assign(collinearLine, solveLinearEquation(
        {x: collinearLine.p1!.left, y: collinearLine.p1!.top}, {x: collinearLine.p2!.left, y: collinearLine.p2!.top}
      ))
      collinearLine.set({
          x1: collinearLine.p1!.left! - COLL_OFF_SET,
          y1: collinearLine.m! * (collinearLine.p1!.left! - COLL_OFF_SET) + collinearLine.b!,
          x2: collinearLine.p2!.left! + COLL_OFF_SET,
          y2: collinearLine.m! * (collinearLine.p2!.left! + COLL_OFF_SET) + collinearLine.b!,
      })
      log("coll", collinearLine)


      // Bind related lines and intersects to points
      pointA.upLine = [aBprime, aCprime]
      pointA.intersects = [pointZ, pointY]
      pointB.upLine = [bAprime, bCprime]
      pointB.intersects = [pointZ, pointX]
      pointC.upLine = [cAprime, cBprime]
      pointC.intersects = [pointY, pointX]
      pointPrimeA.downLine = [bAprime, cAprime]
      pointPrimeA.intersects = [pointZ, pointY]
      pointPrimeB.downLine = [aBprime, cBprime]
      pointPrimeB.intersects = [pointZ, pointX]
      pointPrimeC.downLine = [aCprime, bCprime]
      pointPrimeC.intersects = [pointY, pointX]

      // Bind lines to intersects
      pointZ.crossLines = [aBprime, bAprime]
      pointY.crossLines = [cAprime, aCprime]
      pointX.crossLines = [bCprime, cBprime]

      const onPointMove = (e: IEvent): void => {
        let p = e.target! as Circle
        if (p.upLine != null) {
          p.top = 400 // Hard-coded for now
        } else {
          p.top = upperLine.m! * p.left! + upperLine.b!
        }

        p.upLine && p.upLine.forEach(line => line.set({ x1: p.left }))
        p.downLine && p.downLine.forEach(line => line.set({ x2: p.left, y2: p.top }))

        if (p.intersects) {
            p.intersects.forEach(inter => {
              let [l1, l2] = inter.crossLines!
              let intersect = getIntersectFromLines(l1, l2)
              let coord = { left: intersect.x, top: intersect.y }
              inter.set(coord)
              inter.label && setLabelToPoint([inter.label!], [coord])
            })
            Object.assign(collinearLine, solveLinearEquation(
              {x: collinearLine.p1!.left, y: collinearLine.p1!.top}, {x: collinearLine.p2!.left, y: collinearLine.p2!.top}
            ))
            log(collinearLine)
            collinearLine.set({
              x1: collinearLine.p1!.left! - COLL_OFF_SET,
              y1: collinearLine.m! * (collinearLine.p1!.left! - COLL_OFF_SET) + collinearLine.b!,
              x2: collinearLine.p2!.left! + COLL_OFF_SET,
              y2: collinearLine.m! * (collinearLine.p2!.left! + COLL_OFF_SET) + collinearLine.b!,
            })
        }

        p.label && setLabelToPoint([p.label!], [{top: p.top!, left: p.left!}])
      }

      canvas.on("object:moving", onPointMove)

      // aBprime.controls = 

      canvas.add(aLabel, bLabel, cLabel,
        aprimeLabel, bprimeLabel, cprimeLabel, xLabel, yLabel, zLabel)
      canvas.add(...movablePoints)
      canvas.add(aBprime, aCprime, bAprime, bCprime, cAprime, cBprime, collinearLine)
      canvas.add(pointX, pointY, pointZ)
      canvas.add(bottomLine, upperLine)
      // const pB = new fabric.Point(90, 100);
    },
  },
);
</script>