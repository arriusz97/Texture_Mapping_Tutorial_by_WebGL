# WebGL Tutorial

WebGL Texture Mapping Tutorial

(MIT License) 2020 LEE ByeoungChan



물체를 가상의 3D공간에 있는 그대로 구현하려면 엄청난 양의 폴리곤이 필요합니다.

이러한 문제점을 해결하고자, 물체의 질감, 세부적인 요철 등을 표현해내기 위해 평평한 면 위에 이미지를 덧씌우는 눈속임을 통해 이를 효율적으로 구현하는 것을 텍스쳐 매핑 기술이라 합니다.

이 매핑의 물체의 공간 좌표를 계산하고, 텍스쳐 좌표계와 대응시키는 과정을 이론과 코드로만 배우게 되면 이해하기 쉽지 않을 것이라 생각했습니다.
	
따라서 텍스쳐 좌표계를 손 쉽게 수정하고, 실제 모델에 이를 실시간으로 반영시켜 직접 볼 수 있게 하여 이에 대한 기초지식이 전무한 사람도 어느정도 이해를 할 수 있게 만들기 위하여 본 프로젝트를 고안하였습니다.
	
텍스쳐 좌표계의 편집 방식은 3D 그래픽 툴의 보편적인 방식을 참고하였습니다.



## 구현

우선 3D화면, 텍스쳐 좌표계 화면 둘을 보여주기 위해서 두 canvas에서 각각 따로 webgl을 실행시켰습니다.
```java
var gl;
var gl_uv;

......

function initialiseGL(canvas) {
	OpenPage(-1);
    try {
        // Try to grab the standard context. If it fails, fallback to experimental
        gl = canvas.getContext("webgl") || canvas.getContext("experimental-webgl");
        gl.viewport(0, 0, canvas.width, canvas.height);
    }
    catch (e) {
    }

    if (!gl) {
        alert("Unable to initialise WebGL. Your browser may not support it");
        return false;
    }
    return true;
}

......

function initialiseGL_uv(canvas) {
    try {
        // Try to grab the standard context. If it fails, fallback to experimental
        gl_uv = canvas.getContext("webgl") || canvas.getContext("experimental-webgl");
        gl_uv.viewport(0, 0, canvas.width, canvas.height);
    }
    catch (e) {
    }

    if (!gl_uv) {
        alert("Unable to initialise WebGL. Your browser may not support it");
        return false;
    }
    return true;
}

function main() {
    canvas = document.getElementById("helloapicanvas");
	canvas_uv = document.getElementById("UV");
	
    if (!initialiseGL(canvas)) {
        return;
    }
    if (!initialiseGL_uv(canvas_uv)) {
        return;
    }
    if (!initialiseBuffer()) {
        return;
    }
	if (!initialiseBuffer_uv()) {
        return;
    }
	
    if (!initialiseShaders()) {
        return;
    }
	if (!initialiseShaders_uv()) {
        return;
    }
    requestAnimFrame = (function () {
        return window.requestAnimationFrame || window.webkitRequestAnimationFrame || window.mozRequestAnimationFrame ||
			function (callback) {
			    window.setTimeout(callback, 1000, 60);
			};
    })();

    (function renderLoop() {
        if (true) {
			renderScene()
			renderScene_uv()
......
```





그리고 텍스쳐는  base64 인코딩 기법을 적용하여 다른 서버나 Edge브라우저에 의존하는 일이 없도록 하였습니다.
```java
var img_uv= " data:image/png;base64,iVBORw0KGgo......"
var img_cube = " data:image/png;base64,iVBORw0K......"
var img_dice = " data:image/png;base64,iVBORw0K......"
var img_head=" data:image/png;base64,iVBORw0KGg......"
var img_can= " data:image/png;base64,iVBORw0KGg......"
var img_can1= " data:image/png;base64,iVBORw0KG......"
```






다음은 세부적으로 적용된 기술들입니다.

마우스를 통한 시전 변환은 마우스의 이동거리의 차를 계산하여 해당 값만큼 Model matrix를 rotation 시키는 방식을 사용하였습니다.
[https://github.com/davidwparker/programmingtil-webgl/tree/master/0063-3d-cube-mouse-rotate]를 참고하였습니다.
```java
function mousedown(event) {
    var x = event.clientX;
    var y = event.clientY;
    var rect = event.target.getBoundingClientRect();
    // 마우스가 캔버스 위에 있을 경우에만 클릭을 인식한다 
    if (rect.left <= x && x < rect.right && rect.top <= y && y < rect.bottom) {
      lastX = x;
      lastY = y;
      dragging = true;
    }
  }
function mouseup(event) {
    dragging = false;
  }//클릭을 뗀 경우 끝 

function mousemove(event) {
    var x = event.clientX;
    var y = event.clientY;
    if (dragging) {
      var factor = 10/1000;
      dx = factor * (x - lastX);
      dy = factor * (y - lastY);
        
      rotX += dy;
      rotY += dx;//마우스의 이동거리만큼 회전을 더 시킨다
    }
    lastX = x;
    lastY = y;
	//최근 마우스의 x, y좌표를 업데이트
  }
```





다음은 텍스쳐 좌표 캔버스 표현에 관한 기술입니다.
화면은 우선 가로를 x축, 세로를 y축으로 둔 채 회전을 못하도록 고정시켰습니다. 줌 인/아웃은 마우스 휠로 가능합니다.
```java
var view_scale = 1;

......

function mousewheel_uv(event) {
	event.preventDefault();
    if(event.wheelDelta<0){
		view_scale = view_scale-0.02;
	}
	else{
		view_scale = view_scale+0.02;
	}
}
```






그리고 캔버스 위에서 마우스를 클릭 시, 마우스 위치에 충분히  가까운 버텍스를 계산한 후, 해당 버텍스는 클릭이 유지되는 동안 붉은빛으로 render되도록 만들었습니다.
```java
function renderScene_uv() {
    gl_uv.clearColor(1.0, 1.0, 1.0, 1.0);
	gl_uv.clearDepth(1.0);
    
    ......

	for(var i=0; i<Nvertex_uv; i++) {
		if(i==selected_vertex) {
			gl_uv.texImage2D(gl_uv.TEXTURE_2D, 0, gl_uv.RGBA, 1, 1, 0, gl_uv.RGBA, gl_uv.UNSIGNED_BYTE, new Uint8Array([255, 0, 0, 255])); ///선택된 상태인 버텍스 (255,0,0,255)
		}
		else gl_uv.texImage2D(gl_uv.TEXTURE_2D, 0, gl_uv.RGBA, 1, 1, 0, gl_uv.RGBA, gl_uv.UNSIGNED_BYTE, new Uint8Array([166, 166, 166, 255])); ///나머지 버텍스 (166,166,166,255)
		gl_uv.drawArrays(gl_uv.POINT, i, 1);
	}
	return true
}
......

function mousedown_uv(event) {
    var x = event.offsetX;
    var y = event.offsetY;
    var rect = event.target.getBoundingClientRect();
    lastX_uv = x;
    lastY_uv = y;
    dragging_uv = true;
    
	for(var i=0; i<Nvertex_uv; i++) {
		if( Math.abs(((ver[i][0]-0.5)*screen_uv_W*view_scale)-lastX_uv+300)<10 &&  Math.abs(((0.5-ver[i][1])*screen_uv_H*view_scale)-300+lastY_uv)<10){
			selected_vertex = i;//클릭 당시 가장 가까운 버텍스
		}
    }
  }
  function mouseup_uv(event) {
    dragging_uv = false;
	selected_vertex = -1;
	
}
    
```






선택된 버텍스를 이동시키는 것은 위에서 참고한 마우스 시점변환 알고리즘을 변형하여 적용하였습니다.
```java
function mousemove_uv(event) {
    var x = event.offsetX;
    var y = event.offsetY;
    if (dragging_uv && selected_vertex!=-1) {
	
      var factor = 1/view_scale;
      var dx_uv = factor/screen_uv_W * (x - lastX_uv);
      var dy_uv = factor/screen_uv_H * (y - lastY_uv);
      ver[selected_vertex][0] = ver[selected_vertex][0] + dx_uv;
      ver[selected_vertex][1] = ver[selected_vertex][1] + dy_uv;
    }
    lastX_uv = x;
    lastY_uv = y;
}
```






그리고 양측의 캔버스 모두 장면을 render할 때 마다 갱신된 vertex attribute를 계속 넘겨주는 방식을 사용하여 실시간으로 uv좌표가 업데이트 될 수 있게 하였습니다.
```java
function renderScene() {///좌측 모델의 캔버스
	
    gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(model(1,1,1)), gl.STATIC_DRAW);///텍스쳐가 적용된 3d모델 
    ......
}


function renderScene_uv() {///우측 텍스쳐 좌표계의 캔버스 
    ......
    gl_uv.bufferData(gl_uv.ARRAY_BUFFER, new Float32Array(Bg_uv(ent_map)), gl_uv.STATIC_DRAW);///배경에 놓아지는 텍스쳐 맵
    ......
    gl_uv.bufferData(gl_uv.ARRAY_BUFFER, new Float32Array(uv_map_ver()), gl_uv.STATIC_DRAW);///버텍스사이 이어주는 line들
    ......
    gl_uv.bufferData(gl_uv.ARRAY_BUFFER, new Float32Array(uv_map_ver_sel()), gl_uv.STATIC_DRAW);///버텍스들
    ......
}
```








마지막으로 모델에 추가로 와이어프레임과 버텍스를 보여주는 기능은 이전과 같이 추가적인 drawArrays로 구현하였습니다.
```java
	gl.drawArrays(gl.TRIANGLES, 0, Ntri*3); 

	
	//////와이어프레임
	if(showWire){
		gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, 1, 1, 0, gl.RGBA, gl.UNSIGNED_BYTE, new Uint8Array([255, 255, 255, 255]));
		
		for(var i=0; i<Ntri; i++) {
			gl.drawArrays(gl.LINE_LOOP, i*3, 3); 
		}
	}
	
	
	//////선택된 버텍스 표현
	if(showVer){
		gl.bufferData(gl.ARRAY_BUFFER, new Float32Array(model_ver(1,1,1)), gl.STATIC_DRAW);
		gl.bindBuffer(gl.ARRAY_BUFFER, gl.vertexBuffer);
		gl.enableVertexAttribArray(0);
		gl.vertexAttribPointer(0, 3, gl.FLOAT, gl.FALSE, 12, 0);	
		

		for(var i=0; i<Nvertex_uv; i++) {
			if(i==selected_vertex) {
				gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, 1, 1, 0, gl.RGBA, gl.UNSIGNED_BYTE, new Uint8Array([255, 0, 0, 255]));
				///선택된 상태인 버텍스 (255,0,0,255)
			}
			else gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, 1, 1, 0, gl.RGBA, gl.UNSIGNED_BYTE, new Uint8Array([0, 0, 0, 255]));
				///나머지 버텍스 (0,0,0,255)
			//gl_uv.drawElements(gl_uv.POINT, 1, gl_uv.UNSIGNED_SHORT,i*2);
			gl.drawArrays(gl.POINT, i, 1);
		}	
	}
```



## EXTRA
```python
import numpy as np
folder = '오브젝트 파일 경로/'
f = open(folder+"오브젝트 파일 이름.obj", 'r')
lines = f.readlines()


Nver = 0;
Nuv = 0;
Nmatch = 0;
Npair = 0;
Nuvtri = 0;
Ntri = 0;

for line in lines:
    items = line.split(" ")
    for item in items:
        if(item == "v"):
            if(Nver == 0): verList = np.array([items[2]+', '+items[3]+', '+items[4].replace('\n',', ')])
            else: verList = np.append(verList,[items[2]+', '+items[3]+', '+items[4].replace('\n',', ')], axis=0)
            Nver +=1
        elif(item == "vt"):
            if(Nuv == 0): uvList = np.array(['['+items[1]+', '+str(1-float(items[2]))+'], '])
            
            else: uvList = np.append(uvList,['['+items[1]+', '+str(1-float(items[2]))+'], '], axis=0)
            
            Nuv +=1
        elif(item == "f"):
            if(len(items) == 5):
                Ntri +=1;
                if(Nmatch == 0): 
                    temp_model = np.array([items[1]])
                    temp_model = np.append(temp_model, items[2])
                    temp_model = np.append(temp_model, items[3].replace('\n',' '))
                else:
                    temp_model = np.append(temp_model, items[1])
                    temp_model = np.append(temp_model, items[2])
                    temp_model = np.append(temp_model, items[3].replace('\n',' '))
            
            else:
                Ntri +=2;
                if(Nmatch == 0): 
                    temp_model = np.array([items[1]])
                    temp_model = np.append(temp_model, items[2])
                    temp_model = np.append(temp_model, items[3])
                    temp_model = np.append(temp_model, items[4].replace('\n',' '))
                    temp_model = np.append(temp_model, items[1])
                    temp_model = np.append(temp_model, items[3])
                else:
                    temp_model = np.append(temp_model, items[1])
                    temp_model = np.append(temp_model, items[2])
                    temp_model = np.append(temp_model, items[3])
                    temp_model = np.append(temp_model, items[4].replace('\n',' '))
                    temp_model = np.append(temp_model, items[1])
                    temp_model = np.append(temp_model, items[3])
            Nmatch +=1
        break;


for pair in temp_model:
    item = pair.split("/")
    if(Npair==0): 
        
        pair_model = np.array([int(item[0])-1,int(item[1])-1])
    else:  
        pair_model = np.append(pair_model, [int(item[0])-1,int(item[1])-1],axis=0)
    Npair +=1;
pair_model = pair_model.reshape(-1,2)
print(Npair)
Npair=0


paircount = 0;

for pair in pair_model:
    if(Npair==0): 
        model = np.array([verList[pair[0]]+' 1.0, 0.0, 0.0, 1.0, ver['+str(pair[1])+'][0], ver['+str(pair[1])+'][1],'])
        uv_map_ver = np.array(['(ver['+str(pair[1])+'][0]-0.5)*2, -(ver['+str(pair[1])+'][1]-0.5)*2,	-1.0,'])
        model_ver = np.array([verList[pair[0]]])
    else:  
        model = np.append(model, [verList[pair[0]]+' 1.0, 0.0, 0.0, 1.0, ver['+str(pair[1])+'][0], ver['+str(pair[1])+'][1],'],axis=0)
        uv_map_ver = np.append(uv_map_ver, ['(ver['+str(pair[1])+'][0]-0.5)*2, -(ver['+str(pair[1])+'][1]-0.5)*2,	-1.0,'],axis=0)
       
    if(paircount < pair[1]):
        model_ver = np.append(model_ver,[verList[pair[0]]],axis=0)
        paircount = pair[1]
        
        
    Npair +=1;
print (paircount)

for uvnum in range(Nuv):
    Nuvtri +=1;
    if(uvnum==0):
        uv_map_ver_sel = np.array(['(ver['+str(uvnum)+'][0]-0.5)*2, -(ver['+str(uvnum)+'][1]-0.5)*2,	-1.1, '])
    else:
        uv_map_ver_sel = np.append(uv_map_ver_sel, ['(ver['+str(uvnum)+'][0]-0.5)*2, -(ver['+str(uvnum)+'][1]-0.5)*2,	-1.1, '],axis=0)



ftemp = open(folder+"vertex.txt", 'w')
for i in verList:
    ftemp.write(i)
ftemp.close()    

ftemp = open(folder+"ver_uv.txt", 'w')
for i in uvList:
    ftemp.write(i)
ftemp.close()    
    
ftemp = open(folder+"model.txt", 'w')
for i in model:
    ftemp.write(i)
ftemp.close()   
    
ftemp = open(folder+"uv_map_ver.txt", 'w')
for i in uv_map_ver:
    ftemp.write(i)
ftemp.close()   

ftemp = open(folder+"uv_map_ver_sel.txt", 'w')
for i in uv_map_ver_sel:
    ftemp.write(i)
ftemp.close()   


ftemp = open(folder+"model_ver.txt", 'w')
for i in model_ver:
    ftemp.write(i)
ftemp.close()   

ftemp = open(folder+"number of vertex.txt", 'w')
ftemp.write("Nvertex_uv = "+ str(Nuvtri) + "\n Ntri = "+ str(Ntri))
ftemp.close()   



```
오브젝트 파일을 제 코드에서 바로 사용 가능한 array의 형태로 만들어주는 파이썬 코드입니다.
다른 참고자료 없이 직접 작성하였습니다.



![](img/out.jpg)
3ds max에서 obj파일로 저장 시 사용한 옵션입니다.





## References
- https://github.com/davidwparker/programmingtil-webgl/tree/master/0063-3d-cube-mouse-rotate (마우스로 큐브 회전시키기)
- https://www.solarsystemscope.com/ (지구 texture)
