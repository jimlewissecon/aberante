# aberante
<?php

use Illuminate\Support\Facades\Route;
use Illuminate\Http\Request;
use Illuminate\Support\Str;
use Illuminate\Support\Facades\Broadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Broadcasting\Channel;

/*
|--------------------------------------------------------------------------
| TABLE UNIQUE (rooms)
|--------------------------------------------------------------------------
*/

Schema::create('rooms', function ($table) {
    $table->id();
    $table->string('code')->unique();
    $table->integer('p1_time')->default(300);
    $table->integer('p2_time')->default(300);
    $table->tinyInteger('turn')->default(1);
    $table->timestamp('last_update')->nullable();
    $table->timestamps();
});

/*
|--------------------------------------------------------------------------
| EVENT REALTIME
|--------------------------------------------------------------------------
*/

class TimerUpdated implements ShouldBroadcast
{
    public $room;

    public function __construct($room)
    {
        $this->room = $room;
    }

    public function broadcastOn()
    {
        return new Channel('room.' . $this->room->code);
    }

    public function broadcastAs()
    {
        return 'timer.updated';
    }
}

/*
|--------------------------------------------------------------------------
| ROUTES (FULL APP)
|--------------------------------------------------------------------------
*/

Route::get('/', function () {
    return redirect('/create');
});

/* CREATE ROOM */
Route::get('/create', function () {

    $room = DB::table('rooms')->insertGetId([
        'code' => strtoupper(Str::random(6)),
        'created_at' => now(),
        'updated_at' => now(),
    ]);

    $r = DB::table('rooms')->where('id', $room)->first();

    return redirect("/room/".$r->code);
});

/* SHOW ROOM */
Route::get('/room/{code}', function ($code) {

    $room = DB::table('rooms')->where('code', $code)->first();

    return view('room', compact('room'));
});

/* SWITCH TURN (CORE LOGIC) */
Route::post('/room/{code}/switch', function ($code) {

    $room = DB::table('rooms')->where('code', $code)->first();

    $now = now();

    if ($room->last_update) {
        $elapsed = $now->diffInSeconds($room->last_update);

        if ($room->turn == 1) {
            $room->p1_time = max(0, $room->p1_time - $elapsed);
        } else {
            $room->p2_time = max(0, $room->p2_time - $elapsed);
        }
    }

    $room->turn = $room->turn == 1 ? 2 : 1;

    DB::table('rooms')
        ->where('code', $code)
        ->update([
            'p1_time' => $room->p1_time,
            'p2_time' => $room->p2_time,
            'turn' => $room->turn,
            'last_update' => $now,
            'updated_at' => now()
        ]);

    broadcast(new TimerUpdated($room));

    return response()->json($room);
});

/*
|--------------------------------------------------------------------------
| VIEW (ONE PAGE UI)
|--------------------------------------------------------------------------
*/
?>

<!DOCTYPE html>
<html>
<head>
    <title>SyncTimer SaaS</title>

    <script src="https://cdn.jsdelivr.net/npm/axios/dist/axios.min.js"></script>

    <style>
        body {
            background: #0f172a;
            color: white;
            font-family: Arial;
            text-align: center;
            padding-top: 80px;
        }

        .timer {
            font-size: 70px;
            margin: 30px;
        }

        button {
            padding: 15px 30px;
            font-size: 18px;
            background: #22c55e;
            border: none;
            border-radius: 10px;
            cursor: pointer;
        }

        .active {
            color: #22c55e;
        }
    </style>
</head>

<body>

<h1>ROOM: {{ $room->code }}</h1>

<div id="p1" class="timer">05:00</div>
<div id="p2" class="timer">05:00</div>

<button onclick="switchTurn()">SWITCH TURN</button>

<script>
let p1 = {{ $room->p1_time }};
let p2 = {{ $room->p2_time }};
let turn = {{ $room->turn }};
let code = "{{ $room->code }}";

function format(s){
    return Math.floor(s/60)+":"+String(s%60).padStart(2,'0');
}

function render(){
    document.getElementById("p1").innerText = format(p1);
    document.getElementById("p2").innerText = format(p2);
}

render();

/*
|--------------------------------------------------------------------------
| REALTIME SIMULATION (simple fallback SaaS version)
|--------------------------------------------------------------------------
*/

setInterval(() => {

    if (turn === 1) p1--;
    else p2--;

    render();

}, 1000);

/*
|--------------------------------------------------------------------------
| SWITCH TURN
|--------------------------------------------------------------------------
*/

function switchTurn(){

    axios.post(`/room/${code}/switch`)
        .then(res => {
            p1 = res.data.p1_time;
            p2 = res.data.p2_time;
            turn = res.data.turn;
            render();
        });

}

</script>

</body>
</html>mimuteurht.com
