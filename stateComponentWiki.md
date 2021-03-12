# State Component and Entity State

The state component can be added to entities to provide an API that manages entity states and their transitions to other states. This API consists of two parts:

* Entity State
```java
EntityState initialEntityState = new EntityState("stateName");
```

* State Component
```java
someEntity.addComponent(new StateComponent(initialEntityState));
```


# Example: Enemy AI

When designing the actions of an enemy in a game, it may be helpful to manage state in an separate component. An entity state can override methods in the `EntityState` class to handle an enemies actions in a specific state and `StateComponent` can help to manage the enemy's behavior and switch states based on parameters like distance to other entities, health, or speed.

A state component is attached to an entity and is composed of one or more entity states. An entity object created without a specified name is given the name `"DEFAULT_IDLE"`. In this example, the enemy will be in one of four states: 

* `"PATROL"`
* `"ATTACK"`
* `"FLEE"`
* `"DEAD"`


## Define Entity with StateComponent

We can start by defining our enemy entity in an entity factory and attaching `StateComponent` and `EnemyComponent`:

```java
@Spawns("enemy")
public Entity newEnemy(SpawnData data) {

    return FXGL.entityBuilder()
            .viewWithBBox(new Rectangle(20,20, Color.RED))
            .with(new StateComponent())
            .with(new EnemyComponent())
            .build();
}
```


## Creating EnemyComponent to Manage State

Now that our enemy can be built, we can manage state by defining our `EnemyComponent` class. Our enemy will start with full health and some speeds defined for each state.

```java
public class EnemyComponent extends Component {

    public int health = 100;

    private Entity player;
    private StateComponent state;

    private final int patrolSpeed = 40;
    private final int attackSpeed = 60;
    private final int fleeSpeed = 70;
    private final int returnSpeed = 200;

}
```

Overriding the `onAdded()` method of our `EnemyComponent` class, we can grab the state component when the enemy entity is attached to the game world and start its patrol. This component also needs access to the player to measure the distance from the enemy and switch states accordingly.

```java
@Override
public void onAdded() {
    state = entity.getComponent(StateComponent.class);
    player = FXGL.getGameWorld().getEntitiesByType(EntityTypes.PLAYER).get(0);

    state.changeState(PATROL);
}
```


## Defining Enemy States

In our patrol state we can define our entity's speed specific to the state it is returning from. Our enemy will return to its patrol quickly after attacking. The `onUpdate()` method defines the path of our enemy's patrol.

```java
private final EntityState PATROL = new EntityState("PATROL") {
        private String direction = "left";

        @Override
        public void onEntering() {
            speed = patrolSpeed;
        }

        @Override
        public void onEnteredFrom(EntityState previousState) {
            if (nextState == ATTACK) {
                speed = returnSpeed;
            }
        }

        @Override
        protected void onUpdate(double tpf) {
            if (entity.getX() < 50) {
                // Move entity continuously left
                direction = "right";

            } else if (entity.getX() > 100){
                // Move entity right
                direction = "left";

            } else if (direction == "left") {
                // Move entity left
            } else {
                // Move entity right
            }
        }
    };
```

The attack state also adjusts the enemy's speed and the enemy will follow the player until it is close enough to attack.

```java
private final EntityState ATTACK = new EntityState("ATTACK") {
    @Override
    public void onEntering() {
        speed = attackSpeed;
    }

    @Override
    protected void onUpdate(double tpf) {

        if (entity.distance(player) > 20) {
            // Move enemy towards player
        }
        else {
            // Enemy attack
        }
    }
};
```

The flee state moves the enemy further from the player.

```java
private final EntityState FLEE = new EntityState("FLEE") {
    @Override
    public void onEntering() {
        speed = fleeSpeed;
    }

    @Override
    protected void onUpdate(double tpf) {
        // Move enemy away from player
    }
};
```

When the enemy is dead we can change its view to reflect that and stop its movement.

```java
private final EntityState DEAD = new EntityState("DEAD") {
    @Override
    public void onEntering() {
        speed = 0;
        entity.getViewComponent().clearChildren();
        entity.getViewComponent().addChild(new Rectangle(20, 20, Color.BLACK));
    }
};
```


## Define State Transition Conditions

Finally, we can override the `onUpdate()` method of `EnemyComponent` to manage our state transitions. If our enemy is dead, it will stay dead, otherwise, it will flee if its health is low enough. In other states it will patrol the pattern defined in the patrol state's `onUpdate()` method. However, if the enemy and player are close enough, the enemy will approach the player and attack instead.

```java
@Override
public void onUpdate(double tpf) {
    if (health <= 0) { 
        state.changeState(DEAD); 
    }
    else if (health < 20) {
        state.changeState(FLEE);
    }
    else if (entity.distance(player) < 150) {
        state.changeState(ATTACK);
    } 
    else {
        state.changeState(PATROL);
    }
}
```