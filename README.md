<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>疯狂卡丁对决Pro - 双人联机版</title>
    <script src="https://cdn.socket.io/4.7.2/socket.io.min.js"></script>
    <style>
        body{margin:0;background:#000;overflow:hidden;font-family:Arial;}
        canvas{background:linear-gradient(#87CEEB,#228B22);border-radius:15px;box-shadow:0 10px 40px rgba(0,0,0,0.8);}
        #ui{position:absolute;top:10px;left:10px;color:gold;font-size:20px;}
        #coins{position:absolute;top:10px;right:10px;color:gold;font-weight:bold;}
        #controls{position:absolute;bottom:20px;left:20px;right:20px;display:flex;justify-content:space-between;}
        .joy{width:90px;height:90px;background:rgba(255,255,255,0.25);border-radius:50%;position:relative;}
        .knob{width:45px;height:45px;background:#FF5722;border-radius:50%;position:absolute;top:50%;left:50%;transform:translate(-50%,-50%);}
        button{width:80px;height:80px;border-radius:50%;border:none;background:#4CAF50;color:white;font-size:16px;font-weight:bold;}
        #ad{position:absolute;top:50%;left:50%;transform:translate(-50%,-50%);background:white;padding:30px;border-radius:20px;display:none;text-align:center;}
    </style>
</head>
<body>
    <canvas id="c"></canvas>
    <div id="ui">点击左边创建房间 | 右边加入房间</div>
    <div id="coins">金币: <span id="coin">500</span></div>
    <div id="controls" style="display:none">
        <div class="joy" id="joy"><div class="knob" id="knob"></div></div>
        <div>
            <button id="skill">技能</button>
            <button id="adbtn">广告</button>
        </div>
    </div>
    <div id="ad"><h3>广告加载中...</h3><div id="timer">3</div></div>

    <script>
        // ====== 所有游戏逻辑（已精简到最小可运行版）======
        const canvas=document.getElementById('c'),ctx=canvas.getContext('2d');
        const socket=io('https://crazy-kart-server.glitch.me'); // 第三步改这里！
        let game={state:'menu',room:null,id:null,coins:localStorage.coins||500,role:0,progress:0,x:200,y:500,vx:0,vy:0,energy:3,items:[]};
        const roles=[{name:"闪电侠",color:"#F44336",skill:"5秒双倍速"},{name:"坦克",color:"#2196F3",skill:"10秒无敌"}];
        localStorage.coins=game.coins;

        function resize(){canvas.width=innerWidth;canvas.height=innerHeight;}
        resize();addEventListener('resize',resize);

        // 菜单点击
        canvas.onclick=e=>{
            if(game.state==='menu'){
                if(e.clientX<innerWidth/2){
                    socket.emit('create'); // 创建房间
                }else{
                    let code=prompt("输入房间号(4位大写)");
                    if(code)socket.emit('join',code);
                }
            }
        };

        // 摇杆
        let tid=null,angle=0,dist=0;
        document.getElementById('joy').ontouchstart=e=>{e.preventDefault();tid=e.changedTouches[0].identifier;}
        document.ontouchmove=e=>{e.preventDefault();
            for(let t of e.changedTouches){
                if(t.identifier===tid){
                    let rect=document.getElementById('joy').getBoundingClientRect();
                    let dx=t.clientX-(rect.left+45), dy=t.clientY-(rect.top+45);
                    dist=Math.min(40,Math.hypot(dx,dy));
                    angle=Math.atan2(dy,dx);
                    document.getElementById('knob').style.transform=`translate(${dist*Math.cos(angle)}px,${dist*Math.sin(angle)}px)`;
                    game.vx=dist*Math.cos(angle)/10;
                    if(Math.abs(angle)<1)game.vy=-10; // 上滑跳
                }
            }
        };
        document.ontouchend=e=>{
            if(e.changedTouches[0]?.identifier===tid){
                dist=0;document.getElementById('knob').style.transform='translate(-50%,-50%)';
            }
        };

        // 技能 & 广告
        document.getElementById('skill').onclick=()=>{if(game.energy>=3){socket.emit('skill');game.energy-=3;}};
        document.getElementById('adbtn').onclick=()=>{showAd(()=>game.coins=(localStorage.coins=game.coins*1+200)*1);};

        function showAd(cb){
            let t=3;document.getElementById('ad').style.display='block';
            let iv=setInterval(()=>{document.getElementById('timer').textContent=--t;if(t<=0){clearInterval(iv);document.getElementById('ad').style.display='none';cb();}},1000);
        }

        // Socket事件
        socket.on('created',id=>{game.room=id;game.state='wait';document.getElementById('ui').innerHTML=`房间号: ${id}<br>发给朋友加入！`;});
        socket.on('joined',()=>{game.state='race';document.getElementById('controls').style.display='flex';document.getElementById('ui').innerHTML='开始比赛！';});
        socket.on('state',s=>{/* 接收其他玩家位置等 */});

        // 主循环
        (function loop(){
            ctx.clearRect(0,0,canvas.width,canvas.height);
            // 简单背景
            ctx.fillStyle='#90EE90';ctx.fillRect(0,canvas.height-100,canvas.width,100);
            // 自己的车
            ctx.fillStyle=roles[game.role].color;
            ctx.fillRect(game.x-25,game.y-15,50,30);
            // 更新位置
            game.x+=game.vx; game.y+=game.vy; game.vy+=0.5;
            if(game.y>canvas.height-120){game.y=canvas.height-120;game.vy=0;}
            game.progress+=0.3; // 自动前进
            document.getElementById('coin').textContent=game.coins;
            requestAnimationFrame(loop);
        })();
    </script>
</body>
</html>
