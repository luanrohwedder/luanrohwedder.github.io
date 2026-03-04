---
title: Fazendo clone de Vampire Survivors no Godot - Parte 1
date: 2026-03-04
categories: [Gamedev, Godot]
tags: [gamedev, godot, clone]
image: /assets/img/2026-03-04/vampire-survivors.jpg
---

Quanto tempo que não escrevo por aqui, última vez foi atualizando sobre o sand simulator usando o [Raylib]({% post_url 2025-10-12-sand-sim-2 %}). Desde então não aconteceu muita coisa, após terminar o projeto, tentei fazer Tetris utilizando Raylib, até consegui fazer o jogo funcionar, mas não tive vontade o suficiente para seguir em frente com ele. Após desistir do Tetris, comecei outro projeto para aprender C# com o framework Avalonia, dessa vez tentando fazer um Gerenciador de RPGs, mas novamente, me vi sem vontade de continuar o projeto, apesar de ter progredido bem e conseguido resolver várias coisas.

Esse ano decidir voltar a tentar fazer jogos novamente (apesar de ter feito o Sand Simulator ano passado), deixando de lado o Raylib, que um dia ainda pretendo voltar, e trabalhar com algo mais "fácil", uma engine pronta. Para isso, decidi usar o [Godot](https://godotengine.org/), uma engine open-source.

Nessa primeira parte, vou relatar como foi o desenvolvimento dos inimigos e dos poderes do jogo, como funciona um Resource, e como eu projetei a minha arquitetura do jogo para expandir sem dificuldades o jogo.

## Introduzindo o Godot

Meu primeiro contato com engine, foi o Construct 3, que usa abordagem de codar por blocos, sem códigos. Fiz meus dois primeiros jogos de game jam nessa engine. Depois fui introduzido a Unity, que foi a engine que mais tive contato, e apesar disso, não desenvolvi nenhum jogo para jams. Mas uma vez que você se acostuma com uma ferramente, é difícil trocar ela, e foi assim comigo ao trocar a Unity para o Godot.

O Godot trabalha inteiramente com *nodes*, tudo é um *node* (até onde eu entendo da engine). Minha maior dificuldade foi acostumar com esse conceito, mas uma vez acostumado, foi questão de ficar pesquisando tutoriais de como fazer as coisas.

![Nodes](/assets/img/2026-03-04/nodes.png)
_Cena do player, cada objeto na imagem é um node_

Dessa forma, utilizando vários *nodes*, podemos criar uma cena, onde cada cena age de forma de independente, é como se fosse seu próprio jogo ali, uma cena de fato. Na imagem acima, temos a cena player constituído com vários nós, caso eu decida rodar apenas essa cena, eu consigo controlar o personagem, de forma independente. Godot basicamente funciona com esses dois conceitos, criamos uma cena com vários nós, e posteriormente, usamos essa cena e outra cena, formando o jogo completo

## Criando a cena do Player

A primeira cena que eu criei foi o jogador, e como é um clone de Vampire Survivors, ele precisa se movimentar em 8 direções. No Godot, tem um node especialmente para corpos que se movimentam, o *CharacterBody2D*. Para ele funcionar, é necessário um colisor dentro dele, para detectar contatos físicos com outros corpos. Nessa primeira parte, o jogador tem apenas uma colisão simples, um *Sprite2D* com textura placeholder, uma *Area2D* que tem outro colisor dentro, servindo para detectar apenas o dano, e por fim, o **PowerManager**, que é outro node e será descrito posteriormente. Dessa forma, temos a cena do Player, o que nos resta é programar a sua movimentação e o dano recebido por contato de outros corpos.

```
extends CharacterBody2D

@export var life : float = 50
@export var speed : float = 150

func _physics_process(delta: float) -> void:
	var direction = Input.get_vector("left", "right", "up", "down")
	velocity = direction * speed
	move_and_slide()

func take_damage() -> void:
	$Hurtbox.set_deferred("monitoring", false)
	$CollisionShape2D.set_deferred("disabled", true)
	await get_tree().create_timer(0.5).timeout
	$Hurtbox.set_deferred("monitoring", true)
	$CollisionShape2D.set_deferred("disabled", false)

func _on_hurtbox_body_entered(body: Node2D) -> void:
	take_damage()
```

Perceba que o Godot tem uma função muito útil do Input que pega a direção do vetor baseado no nosso input, assim ficando muito fácil fazer a movimentação do jogador. Depois de pegar o vetor da direção do input, basta aplicar essa direção a velocidade do *CharacterBody2D* multiplicando pela velocidade, e então chamar a função *move_and_slide()*, que move o jogador e lida com todos os processos de colisão e física.

Agora para receber o dano, precisamos criar um *signal*, que é um sinal que o node envia caso tal ação do sinal aconteça. Nesse caso, se algum corpo entrar na área de dano, ele envia um sinal, e esse sinal deve ser executado. Quando o sinal de dano é recebido, o jogador desativa todas as colisões (tanto do CharacterBody2D e do Hurtbox), cria um timer de 0.5 segundos, o tempo de invencibilidade, e então volta com todos colisores, pronto para receber o dano novamente. Por enquanto não foi implementado reduzir a vida do jogador, estou deixando para quando começar a mexer com a UI do jogo.

Na cena do jogador, é interessante observar alguns conceitos da engine, como a função *_physic_process*, que é feito para lidar com atualizações que envolvem física, como o próprio nome sugere. A anotação de *@export*, que é utilizado para observar e definir o valor da variável no Inspector do node onde o script está ligado.

## Um pouco sobre Resources

Antes de mostrar a próxima cena, é de extrema importância explicar o conceito dos *Resources* do Godot. De forma simplificada, até porque eu mesmo não compreendo muito bem ele de forma detalhada, é um container de dados, ou seja, você pode especificar dados que certo objeto vai conter, e todo objeto que herdar esse resource, vai ter como base esses dados. Então como exemplo, vou mostrar o resource dos inimigos que foi criado para esse jogo.

- Primeiro, criamos um script com dados dados que queremos dos inimigos, ou seja, todos os inimigos herdarão esses mesmos dados. O script deve extender a classe *Resources* para aparecer como um 
- Em sequência, após criar o script, criamos um resource na engine do inimigo em específico que já queremos
- Por fim, criamos uma cena base do inimigo, que nesse caso, irá ter o mesmo comportamento para todos (apenas movimentar até o jogador, variando apenas a vida, velocidade e textura)

```
class_name EnemyStats
extends Resource

@export var health : int
@export var speed : float
@export var texture : Texture2D
```
_Resource base para todos os inimigos_

Veja que o resource criado contém vida, velocidade e textura. Dessa forma, basta criar resource na engine e especificar os valores para cada inimigo. Dessa forma, podemos criar inimigos apenas criando mais resources, sem precisar criar uma cena para cada tipo de inimigo, economizando tempo e recursos.

![Resources](/assets/img/2026-03-04/resources.png)
_Resource criado para o inimigo zumbi_

## Criando a cena do inimigo e do spawn de inimigos

### Inimigo

Como comentando anteriormente, o inimigo desse jogo tem apenas o comportamento de seguir o jogador, diferenciando apenas nos dados contido no resource criado para aquele inimigo em específico. Dito isso, a cena do inimigo é bem simples, precisamos buscar a cena do jogador, inicializar a cena com os dados do resource e então atualizar o movimento dele para o player, além de claro, uma função para receber dano.

![Enemy](/assets/img/2026-03-04/enemy.png)
_Estrutura da cena do inimigo_

Em questão de estrutura, o inimigo é parecido com o player, contendo um *CharacterBody2D* com um *ColliderShape2D* e *Sprite2D* dentro dele. Já para o script, temos uma variável para especificar o resource que o inimigo irá usar, assim como variáveis locais para copiar os dados da resource, no meu caso, por agora tenho apenas a vida atual do inimigo como variável. Eu inicio copiando os dados do resource para cena e busco o player. 

Para atualizar o movimento apenas pego a posição global do player e atualizo o inimigo para seguir ele na velocidade do resource. Para atualizar o estado do inimigo, isto é, se ele deve ser excluído da cena, uso o _*process*, que é usado em casos que não envolve física. Eu verifico se o inimigo está muito distante do player e se sua vida chegou a zero, caso uma dessas condições aconteça, o inimigo é excluído da cena.

E para finalizar, o inimigo possui um método para receber dano, que irá ser chamado pelos poderes. Isso é uma estratégia de arquitetura, de forma que o inimigo não precise conhecer nenhum poder para receber dano, e o poder só precisa perguntar para o corpo que ele entrou em contato, se ele contém um método para receber dano.

```
extends CharacterBody2D

var stats : EnemyStats

var player
var current_health

func _ready() -> void:
	$Sprite2D.texture = stats.texture
	current_health = stats.health
	player = get_node("/root/Main/Player")

func _process(delta: float) -> void:
	var dir = player.global_position - global_position
	var dist = dir.length()
	
	if (dist > 600):
		queue_free()

	if (current_health <= 0):
		queue_free()

func _physics_process(delta: float) -> void:
	var dir = player.global_position - global_position
	var dist = dir.length()
	
	if dist > 16:
		velocity = dir.normalized() * stats.speed
	else:
		velocity = -dir.normalized() * stats.speed * 0.5
	move_and_slide()

func take_damage(damage: int) -> void:
	current_health -= damage
```

A função do movimento ainda precisa ser polida e melhorada, mas para o começo agora está funcionando, além de precisar melhorar o tamanho da hitbox do personagem de acordo com a textura que atribuída a ele.

### Spawn de inimigos

O Vampire Survivors se trata de um jogo de horda, portanto, acredito que o Spawn de inimigos é o principal fator para o jogo funcionar bem, é claro, o que deixa divertido é mecânica do jogador, powerups, upgrades, etc., mas falando de questão técnica, e não da parte lúdica do jogo, creio que o spawn tem mais importância. Dito isso, como isso é um material de estudo, irei aplicar tudo o que contém no Vampire Survivors, sem inventar mecânica nessa parte.

Pesquisando, entendi que o spawn funciona em um raio em volta do player, fora do campo de visão do jogador. Além disso, o jogo funciona por hordas de 1 minuto, ou seja, a cada 1 minutos, muda o tipo de inimigo e o comportamento do spawn. Esse é o funcionamento básico do sistema do Vampire Survivors, não fui muito a fundo pesquisar como funciona ainda, pois apenas queria que o inimigo nascesse em volta do player.

Dessa forma, eu precisava de algumas propriedades básicas para o spawn, como o raio e o máximo de inimigos que pode existir no jogo. Além dessas propriedades básicas, é necessário o player, para saber onde ele se encontra, e então calcular o raio de spawn, a cena do inimigo para instanciá-lo e os resource dos inimigos.

```
extends Node

#Enemies
@export_group("Enemy properties")
@export var enemy_scene : PackedScene = preload("res://scenes/enemies/enemy.tscn")
@export var enemy_stats : Array[EnemyStats]

#Properties
@export_group("Properties")
@export var spawn_radius : float
@export var max_enemies : int

var player
var spawn_wait_time = 1

func _ready() -> void:
	player = get_node("/root/Main/Player")

func _on_spawn_timer_timeout() -> void:
	if player == null:
		return
	
	if $Enemies.get_child_count() >= max_enemies:
		return
		
	var angle = randf() * TAU
	var offset = Vector2.RIGHT.rotated(angle) * spawn_radius
	var spawn_pos = player.global_position + offset
	spawn_enemy(enemy_stats[0], spawn_pos)

func spawn_enemy(stats : EnemyStats, position : Vector2) -> void:
	var enemy = enemy_scene.instantiate()
	enemy.stats = stats
	enemy.global_position = position
	$Enemies.add_child(enemy)
```

O inimigo é instanciado a cada vez que o timeout do node timer dentro da cena do spawn é ativado, talvez exista uma estratégia melhor para fazer isso, mas por enquanto, é dessa forma como está funcionando. Ele pega uma posição aleatória no raio em volta do player, e instância o inimigo, nesse caso, com o resource de zumbi, que é o único que tem ainda. Para a próxima parte, vou precisar aprimorar a questão de hordas, o jeito de criar os inimigos (sem ser por timeout talvez).

![Enemy2](/assets/img/2026-03-04/enemy2.png)
_Imagem ilustrando o funcionamento do spawner de inimigos - Lazy drawing_

## Criando os poderes

Aqui foi onde aprendi sobre os *Resources*, então pode ser que a arquitetura pensada para ele não seja dos melhores, mas acredito que para esse projeto, ele serve bem. A ideia foi basicamente ter um resource para o poder, e desse resource, criar outros resources específicos, como projéteis (adaga, magia em linha reta, etc.), efeitos em área, etc., que estendem do poder base. Depois, para cada resource específico, criar uma cena específica para ele. Até então foi implementado apenas o projétil, então essa seção ainda pode sofrer muitas mudanças na próxima parte.

![Power](/assets/img/2026-03-04/power.png)
_Arquitetura de *Resources* dos poderes_

### Poder base

O poder base é um resource bem simples que contém dados que todos os outros poderes tem que possuir, sendo eles nome, dano, tempo de recarga e a sua cena específica

```
class_name Power
extends Resource

@export var name : String
@export var damage : int
@export var cooldown : float
@export var scene : PackedScene
```

### Poder específico - Projétil

Igual o poder base, um poder específico vai conter dados extras necessários para o funcionamento daquele poder. No caso do projétil, foi necessário uma velocidade e textura (que pode ser movido para o poder base).

```
class_name ProjectilePower
extends Power

@export var speed : float
@export var texture : Texture2D
```

A cena do projétil também é bem simples, uma *Area2D* com *CollisionShape2D*, *Sprite2D* e *VisibleOnScreenNotifier2D*, sendo esse último para detectar quando o projétil sai da camera, útil para excluir o projétil. O script também é bem straightforward, aplica a textura para o objeto, no processo de física apenas vai para a direção do input do jogador (definido no power manager). Aqui temos o que foi comentado na seção do inimigo, que é quando o corpo do projétil atinge algum outro corpo, faz a verificação se esse corpo possuí o método de receber dano, caso possua, chama esse método do corpo atingido.

```
extends Area2D

var power : ProjectilePower

func _ready() -> void:
	$Sprite2D.texture = power.texture

func _physics_process(delta: float) -> void:
	var dir = Vector2.RIGHT.rotated(rotation)
	position += dir * power.speed * delta

func _on_body_entered(body: Node2D) -> void:
	if body.has_method("take_damage"):
		body.take_damage(power.damage)
	queue_free()

func _on_visible_on_screen_notifier_2d_screen_exited() -> void:
	queue_free()
```

### Gerenciador de poderes

O gerenciador de poderes é uma cena para **gerenciar** os poderes, que coisa não? Como é um clone do Vampire Survivors, é necessário gerenciar os poderes conforme o jogador vai evoluindo e escolhendo novos poderes. Como ainda não foi implementado as cartas, questão de level up e tudo mais, o gerenciador ainda só funciona como um ativador dos poderes. Ele gerencia o tempo de recarga de todos os poderes, e de acordo com o tempo de recarga, ativa os poderes. 

O ativamento dos poderes é chamado pelo jogador no *_process*, enquanto o cooldown é calculado no *_process* do gerenciador de poderes.

```
extends Node

var powers : Array
var cooldowns := {}

func _process(delta: float) -> void:
	for power in powers:
		cooldowns[power] -= delta
		if cooldowns[power] <= 0:
			cooldowns[power] = 0

func cast(origin: Vector2, direction: Vector2) -> void:
	for power in powers:		
		if power.scene == null:
			continue

		if cooldowns.get(power, 0.0) > 0.0:
			continue

		var instance = power.scene.instantiate()
		instance.global_position = origin
		instance.power = power
		instance.rotation = direction.angle()
		get_tree().current_scene.add_child.call_deferred(instance)
		
		cooldowns[power] = power.cooldown
```

## Resultado

Com essas pequenas coisas feitas, já temos um jogo semi funcional, onde é possível derrotar os inimigos, mas não é possível perder ainda. No entanto, muita coisa fundamental já está pronta, do ponto de vista de arquitetura, acredito que está bem estruturado, e para expandir os inimigos e poderes, vai ser uma tarefa mais simples.

![Game](/assets/img/2026-03-04/game.gif)

## Próximo passo

Para a próxima parte, que definitivamente vai ter, e espero não demorar mais que 1 semana para isso (impossível do jeito que eu sou), eu pretendo introduzir a parte de arte para o jogo, aprender um pouco sobre pixel art e a parte de UI para o jogo, para então começar a trabalhar com as cartas para selecionar o poder.

Todo List para a próxima parte:
- Desenhar pixel art do player, inimigos e poderes (ao menos 1 poder e inimigo)
- Implementar parte da UI
- Melhorar sistema de hordas
