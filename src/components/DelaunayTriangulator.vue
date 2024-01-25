<template>
  <div class="hello">
    <!-- I want a canvas with width equal to 90% of the screen width -->
      <canvas style="border:1px solid #000000;" id="c" @click="canvasClick"/>
  </div>
</template>

<script>

const POINT_RADIUS = 3;

// calculate angles of triangles


// go from triangles

// to triangles + edge list
// use edge list to check for illegal edges, and then generate new edge list
// to triangles again


// design choices:
// - valdi að einfalda mikið kóðann. Halda t.d. ekki um nágranna og alls konar upplysingar um gagnastrúktúrinn af því að það er lower bound á keyrslutímanum í Omega(n^2) hvort sem er.
// - nota bara lista til að representa punkta og þríhyrninga. Og eini staðurinn sem ég geymi floating points er í einum lista sem eru nóðurnar, allt annað bendir í þann lista.
// - hraðaði með því að kíkja bara á þríhyrninga sem "hafa eitthvað með lituðu þríhyrningana að gera".

export default {
  name: 'DelaunayTriangulator',
  props: {
    msg: String
  },
  mounted() {
    this.c = document.getElementById('c');
    this.ctx = this.c.getContext('2d');
    this.setCanvasSize(false);
    this.addPoint([-1, 2]);
    this.addPoint([0.5, -1]);
    this.addPoint([2, 2]);
    this.addTriangle([0,1,2]);
    let addPoints = (i) => {
      // get random numbers in range [-.1, 1.1]
      const x = Math.random()*1.2 - .1;
      const y = Math.random()*1.2 - .1;
      this.addPoint([x,y]);
      this.handlePoint(this.P.length-1, false);
      if(i > 0) addPoints(i-1);
    };
    var startTime = performance.now();
    addPoints(300);
    this.drawTriangulation();
    var endTime = performance.now();

    window.addEventListener('resize', () => this.setCanvasSize(true)); // Update canvas size on window resize

    console.log(`handling points took ${endTime - startTime}`);
  },
  methods: {
      canvasClick(e) {
        const x = e.offsetX/this.canvasWidth;
        const y = e.offsetY/this.canvasHeight;
        const p = [x,y];
        this.pointQueue.push(this.P.length);
        this.addPoint(p);
        if(this.pointQueue.length == 1) this.handlePoint(this.P.length - 1, true);
      },
      handlePoint(pi, color) {
        const p = this.P[pi];
        for(const [i,t] of this.T.entries()) {
          if(this.pointInsideTriangle(p, t)) {
            let nxt = pi;
            const [a,b,c] = t;
            this.addTriangle([a,b,nxt]);
            this.addTriangle([nxt,b,c]);
            this.addTriangle([nxt,c,a]);
            this.T.splice(i, 1);
            break;
          }
        }
        let coloredTriangles = [this.T.length - 1, this.T.length - 2, this.T.length - 3];
        // go through edge list and swap if edge is illegal
        const delay = 2000;
        let fixTriangulation = () => {
          let found = false;
          if(color) this.drawTriangulation(coloredTriangles);
          let edgeList = this.getEdgeList(coloredTriangles);
          let oldT = [...this.T];
          for(const [a,b,t1,t2] of edgeList) {
            const [flip, altt1, altt2] = this.isEdgeIllegal(a,b,this.T[t1],this.T[t2]);
            if(flip) {
              found = true;
              oldT[t1] = altt1;
              oldT[t2] = altt2;
              if(coloredTriangles.indexOf(t1) < 0) coloredTriangles.push(t1);
              if(coloredTriangles.indexOf(t2) < 0) coloredTriangles.push(t2);

              if(color) this.drawLine(this.P[a], this.P[b], 'red');
            }
          }
          this.T = oldT;
          if(found) {
            if(color) setTimeout(() => fixTriangulation(), delay);
            else fixTriangulation();
          }
          else {
            if(color) {
              this.drawTriangulation(coloredTriangles);
              setTimeout(() => {
                this.drawTriangulation();
                this.pointQueue.shift();
                if(this.pointQueue.length > 0) {
                  this.handlePoint(this.pointQueue[0], true);
                }
              }, delay);
            }
          }
        }
        fixTriangulation();
      },
      angleBetweenTwoVectors(p1, p2) {
        return (180/Math.PI)*Math.acos((p1[0]*p2[0]+p1[1]*p2[1])/Math.sqrt((p1[0]**2+p1[1]**2)*(p2[0]**2+p2[1]**2)));
      },
      diff(p1,p2) {
        return [p1[0]-p2[0], p1[1]-p2[1]];
      },
      calcAngles(t) {
        return [
          this.angleBetweenTwoVectors(this.diff(this.P[t[1]],this.P[t[0]]),this.diff(this.P[t[2]],this.P[t[0]])),
          this.angleBetweenTwoVectors(this.diff(this.P[t[2]],this.P[t[1]]),this.diff(this.P[t[0]],this.P[t[1]])),
          this.angleBetweenTwoVectors(this.diff(this.P[t[0]],this.P[t[2]]),this.diff(this.P[t[1]],this.P[t[2]])),
        ]
      },
      getAngleVectors(t1,t2) {
        return this.calcAngles(t1).concat(this.calcAngles(t2)).sort(function(a,b){return a-b});
      },
      arrayGreaterThan(a,b) {
        for(let i = 0; i < a.length; i++) {
          if(a[i] != b[i]) return a[i] > b[i];
        }
      },
      isEdgeIllegal(a,b,t1,t2) {
        const remainingVertexInt1 = t1.filter(x=>![a,b].includes(x))[0];
        const remainingVertexInt2 = t2.filter(x=>![a,b].includes(x))[0];

        // t1 currently has the suborder a->b and t2 has b->a
        const altt1 = [remainingVertexInt1, a, remainingVertexInt2];
        const altt2 = [remainingVertexInt1, remainingVertexInt2, b];
        if(this.pointInsideTriangle(this.P[b], altt1) || this.pointInsideTriangle(this.P[a], altt2)) return [false];
        return [this.arrayGreaterThan(this.getAngleVectors(altt1, altt2), this.getAngleVectors(t1, t2)), altt1, altt2];
      },
      fillTriangle(t,op) {
        // the triangle
        this.ctx.beginPath();
        const [a,b,c] = t;
        this.ctx.moveTo(this.P[a][0]*this.canvasWidth, this.P[a][1]*this.canvasHeight);
        this.ctx.lineTo(this.P[b][0]*this.canvasWidth, this.P[b][1]*this.canvasHeight);
        this.ctx.lineTo(this.P[c][0]*this.canvasWidth, this.P[c][1]*this.canvasHeight);
        this.ctx.closePath();
        this.ctx.fillStyle = 'rgba(255,0,0,'+ op +')';
        this.ctx.fill();
      },
      drawLine(p1, p2, color) {
        this.ctx.beginPath();
        this.ctx.moveTo(p1[0]*this.canvasWidth, p1[1]*this.canvasHeight);
        this.ctx.lineTo(p2[0]*this.canvasWidth, p2[1]*this.canvasHeight);
        this.ctx.lineCap = 'round';
        this.ctx.lineWidth = 1;

        this.ctx.shadowColor = "rgba(0,0,0,.5)";
        this.ctx.shadowBlur = 2;

        if (color==='black') this.ctx.strokeStyle = '#000000';
        else if (color==='red') this.ctx.strokeStyle = '#ff0000'
        // Draw the Path
        this.ctx.stroke();
      },
      drawPoint(p) {
        this.ctx.beginPath();
        this.ctx.arc(p[0]*this.canvasWidth,p[1]*this.canvasHeight, POINT_RADIUS, 0, 2 * Math.PI, false);
        this.ctx.fillStyle = 'black';
        this.ctx.fill();
        this.ctx.lineWidth = 0;
        this.ctx.strokeStyle = '#000000';
        this.ctx.stroke();
      },
      getEdgeList(subset) {
        if(!subset) {
          subset = [];
          for(let i = 0; i < this.T.length; i++) subset.push(i);
        }
        
        const importantvertices = [];
        for(const i of subset) {
          importantvertices.push(...this.T[i]);
        }
        
        const potentialEdges = [];
        let i = 0;
        for(const t of this.T) {
          if(t.filter(x => importantvertices.includes(x)).length > 1){
            potentialEdges.push([t[0],t[1],i]);  
            potentialEdges.push([t[1],t[2],i]);
            potentialEdges.push([t[2],t[0],i]);
          }
          i++;
        }

        let ret = [];
        for(const [a,b,t1] of potentialEdges) {
          if(a < b) {
            for(const [c,d,t2] of potentialEdges) {
              if(b == c && a == d) {
                ret.push([a,b,t1,t2]);
              }
            }
          }
        }
        return ret;
      },
      drawTriangulation(highlightedTri) {
        this.ctx.clearRect(0,0,this.c.width, this.c.height);
        if(highlightedTri) {
          for(const t of highlightedTri) {
            this.fillTriangle(this.T[t],0.5);
          }
        }
        for(const p of this.P) {
          this.drawPoint(p);
        }
        for(const [a,b] of this.getEdgeList()) {
          this.drawLine(this.P[a], this.P[b], 'black');
        }
      },
      addPoint(p) {
        this.P.push(p);
      },
      addTriangle(t) {
        this.T.push(t);
      },
      
      setCanvasSize(isResize) {
        const canvas = this.c;

        // Ensure the canvas context exists before attempting to set its size
        if (canvas && canvas.getContext) {
          //const context = canvas.getContext('2d');
          const screenWidth = window.innerWidth * 0.9; // 90% of the screen width
          const screenHeight = window.innerHeight * 0.9

          // Set canvas size
          canvas.width = screenWidth;
          canvas.height = screenHeight;
          this.canvasWidth = canvas.width;
          this.canvasHeight = canvas.height;
          // Additional styles or drawing operations can be performed here
        }
        if(isResize) this.drawTriangulation();
      },
      pointInsideTriangle(p, t) {
        const [a,b,c] = t;
        return this.toRight(this.P[a], this.P[b], p)
          && this.toRight(this.P[b], this.P[c], p)
          && this.toRight(this.P[c], this.P[a], p);
      },
      toRight(a,b,p) { // check if p is to the right of a->b
        return (p[0] - a[0])*(b[1] - a[1]) - (p[1] - a[1])*(b[0] - a[0]) < 0;
      }
      
  },
  data() {
    return {
      P: [],
      T: [],
      pointQueue: [],
      canvasWidth: 0.9 * window.innerWidth,
      canvasHeight: 0.9 * window.innerHeight
    }
  }

  
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
h3 {
  margin: 40px 0 0;
}
ul {
  list-style-type: none;
  padding: 0;
}
li {
  display: inline-block;
  margin: 0 10px;
}
a {
  color: #42b983;
}
</style>
