---
title: State Design Pattern  
published: true
tags: LLD Design-Patterns State Python Idiomatic
---

<p align="center">
  <img src="{{site.baseurl}}/assets/images/statepattern.png">
</p>

It compiles, passes test cases, works on production, but is it enough?  
A very important key component of any software system is its design. We'll look at one of the ways you can better design your software for maintainability and readability in this post.  
Let's understand **State Design Pattern** with the help of an example problem statement.

## Requirements

Let's say we want to design a single elevator system for a building with N floors (ignore 2 elevators as shown in the diagram).  
Each floor has 2 buttons - UP and DOWN which the users can press based on whether they want to go up or down.

<p align="center">
  <img height="500px" src="{{site.baseurl}}/assets/images/elevator-system.png">
</p>

## Data Structure and Algorithm

It may sound like a simple problem but trust me elevators are complex.  
We'll not focus a lot on the optimising the data structure, and the algorithm here because no matter how bad the time complexities are, a building is not going to have like a gazillion floors.  

Let's get some clarity on how an elevator functions first.  
**An elevator will try to service all the requests in the direction it is moving first and when it is done serving all requests the direction of its movement is when it will change direction and start serving all the other ones.**  

This statement in bold is in fact the most important one to understand.  

## It Compiles and Works (1st Attempt)

At first attempt, I chose max heap to service all requests going in the downward direction, and a min heap to service while going up.  
Like I said, the choice of a data structure isn't that important but just take a rough look at what I wrote when I started implementing it without a plan in head.  
(Pro Tip: Don't waste your time trying to understand what's happening, not worth it!)
```python
import enum
import heapq
import logging
import threading
import time


logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    handlers=[
        logging.FileHandler("log_file.log"),
    ]
)
logger = logging.getLogger(__name__)


class Direction(enum.Enum):
    up = 0
    down = 1
    idle = 2  # elevator has no request to serve


class Elevator:
    def __init__(self, floor: int, direction: Direction):
        self.floor = floor
        self.direction = direction
        logger.info("at floor {}".format(self.floor))

    def go_up_1_floor(self):
        self.floor += 1
        logger.info("at floor {}".format(self.floor))
        time.sleep(1)

    def go_down_1_floor(self):
        self.floor -= 1
        logger.info("at floor {}".format(self.floor))
        time.sleep(1)


class ElevatorSystem:
    def __init__(self):
        self._elevator = Elevator(0, Direction.idle)
        self._requests_can_serve_going_down = []  # max heap
        self._requests_can_serve_going_up = []  # min heap
        self._pending_requests_for_next_round = []  # normal queue

    def _feed_input(self, floor: int, direction: Direction):
        if self._elevator.direction == Direction.down:
            if direction == Direction.down:
                # you can either server it or not
                if self._elevator.floor >= floor >= min(self._requests_can_serve_going_down):
                    self._requests_can_serve_going_down.append(floor)
                    heapq._heapify_max(self._requests_can_serve_going_down)
                else:
                    self._pending_requests_for_next_round.append([floor, direction])
            elif direction == Direction.up:
                self._pending_requests_for_next_round.append([floor, direction])

        elif self._elevator.direction == Direction.up:
            if direction == Direction.up:
                if self._elevator.floor <= floor <= max(self._requests_can_serve_going_up):
                    self._requests_can_serve_going_up.append(floor)
                    heapq.heapify(self._requests_can_serve_going_up)
                else:
                    self._pending_requests_for_next_round.append([floor, direction])
            else:
                self._pending_requests_for_next_round.append([floor, direction])

        else:
            self._pending_requests_for_next_round.append([floor, direction])

    def take_requests_forever(self):
        while True:
            floor = int(input("what floor are you at: "))
            direction = input("which direction do you want to go (u/U/d/D): ")
            if direction in {'u', 'U'}:
                direction = Direction.up
            else:
                direction = Direction.down
            self._feed_input(floor, direction)

    def _serve_pending_requests(self):
        floor_destination, destination_direction = self._pending_requests_for_next_round.pop(0)
        if self._elevator.floor > floor_destination:
            # need to go down
            new_pending_requests = []
            self._requests_can_serve_going_down.append(floor_destination)
            for index, request in enumerate(self._pending_requests_for_next_round):
                if floor_destination <= request[0] <= self._elevator.floor:
                    self._requests_can_serve_going_down.append(request[0])
                else:
                    new_pending_requests.append(request)
            self._pending_requests_for_next_round = new_pending_requests
            heapq._heapify_max(self._requests_can_serve_going_down)
            self._elevator.direction = Direction.down

        elif self._elevator.floor < floor_destination:
            # need to go up
            new_pending_requests = []
            self._requests_can_serve_going_up.append(floor_destination)
            for index, request in enumerate(self._pending_requests_for_next_round):
                if self._elevator.floor <= request[0] <= floor_destination:
                    self._requests_can_serve_going_up.append(request[0])
                else:
                    new_pending_requests.append(request)
            self._pending_requests_for_next_round = new_pending_requests
            heapq.heapify(self._requests_can_serve_going_up)
            self._elevator.direction = Direction.up

        else:
            # at that floor
            logger.info("reached destination {}".format(floor_destination))

    def _serve_forever(self):
        while True:
            if self._elevator.direction == Direction.idle:
                if len(self._pending_requests_for_next_round):
                    self._serve_pending_requests()
                else:
                    time.sleep(1)
            elif self._elevator.direction == Direction.down:
                if len(self._requests_can_serve_going_down):
                    if self._elevator.floor > self._requests_can_serve_going_down[0]:
                        self._elevator.go_down_1_floor()
                    elif self._elevator.floor == self._requests_can_serve_going_down[0]:
                        logger.info("reached destination {}".format(self._requests_can_serve_going_down[0]))
                        heapq._heappop_max(self._requests_can_serve_going_down)
                elif len(self._pending_requests_for_next_round):
                    self._serve_pending_requests()
                elif not len(self._pending_requests_for_next_round):
                    self._elevator.direction = Direction.idle
            elif self._elevator.direction == Direction.up:
                if len(self._requests_can_serve_going_up):
                    if self._elevator.floor < self._requests_can_serve_going_up[0]:
                        self._elevator.go_up_1_floor()
                    elif self._elevator.floor == self._requests_can_serve_going_up[0]:
                        logger.info("reached destination {}".format(self._requests_can_serve_going_up[0]))
                        heapq.heappop(self._requests_can_serve_going_up)
                elif len(self._pending_requests_for_next_round):
                    self._serve_pending_requests()
                elif not len(self._pending_requests_for_next_round):
                    self._elevator.direction = Direction.idle

    def serve_forever(self):
        threading.Thread(target=self._serve_forever, daemon=True).start()


if __name__ == '__main__':
    my_funny_elevator_system = ElevatorSystem()
    my_funny_elevator_system.serve_forever()
    my_funny_elevator_system.take_requests_forever()
```

It compiles, works and serves our purpose. But is there a bug somewhere in here? Maybe, we never know until we encounter it or somebody reads through the entire piece thoroughly.  
Huge method blocks, so many if else statements already make this implementation hard to read through and make changes to.  
After a couple of weeks or months if I have to modify the behaviour, I'll have to understand the entire piece and make relevant changes which will make this piece even more complex than it already is.  


## A Better Design (2nd Attempt)

This is where we take a step back and devise a design strategy before directly diving into writing code.  
If we closely take a look, an Elevator is a Finite State Machine (FSM). It can be in exactly one state at any given instant.  
Because we tried handling logic for each state in our corresponding request methods (`feed_input` and `serve`) is why we had so many if-else spaghetti at first place  

### Identifying States

At any given instant our elevator can be in exactly these 3 states:

1. Going Up State
2. Going Down State
3. Idle State

### Identifying Request Actions

Our Elevator can:

1. take input from user at any floor
2. serve pending requests

(Note: Our Elevator can perform all these actions in parallel)  

### Class Diagram

<p align="center">
  <img height="600px" src="{{site.baseurl}}/assets/images/elevator-state-class-diagram.png">
</p>

This simple design will greatly help us organise our logic and make the entire system more readable and maintainable. Let's deconstruct it piece by piece.  
- We have a common interface/abstract class for a state that our Elevator can be in at any given time. This state is capable of performing all the request actions we earlier discussed.
  ```python
  class ElevatorState(ABC):
    elevator = None
    
    def __init__(self, elevator_):
        self.elevator = elevator_
    
    def set_floor_request_direction(self, floor: int, direction: Direction):
        if direction == Direction.up:
            self.elevator.floor_requests[floor].go_up = True
            return
        self.elevator.floor_requests[floor].go_down = True
    
    @abstractmethod
    def take_request(self, from_floor: int, direction: Direction):
        self.set_floor_request_direction(from_floor, direction)
    
    @abstractmethod
    def serve(self):
        logger.info("Current Floor: %s", self.elevator.current_floor)
    ```
  
- We also have 3 concrete implementations of this abstract `ElevatorState` which corresponds to the 3 states that we earlier identified
    ```python
    class GoingUpState(ElevatorState):
        def serve(self):
            super().serve()
            self.elevator.floor_requests[self.elevator.current_floor].go_up = False
            # if you've reached the target floor then check whether you change direction or go idle
            if self.elevator.current_floor < self.elevator.target_floor:
                self.go_up_one_floor()
            else:
                # time to change state
                floor_to_service_next = self.elevator.find_floor_to_service_from_bottom()
                if not floor_to_service_next:
                    self.elevator.transition_to(self.elevator.idle_state)
                    self.elevator.target_floor = None
                else:
                    self.elevator.transition_to(self.elevator.going_down_state)
                    self.elevator.target_floor = floor_to_service_next
            self.elevator.state.serve()
    
        def take_request(self, from_floor: int, direction: Direction):
            super().take_request(from_floor, direction)
            if direction == Direction.up:
                self.elevator.target_floor = max(self.elevator.target_floor, from_floor)
    
    
    class GoingDownState(ElevatorState):
        def serve(self):
            super().serve()
            self.elevator.floor_requests[self.elevator.current_floor].go_down = False
            # if you've reached the target floor then check whether you change direction or go idle
            if self.elevator.current_floor > self.elevator.target_floor:
                self.go_down_one_floor()
            else:
                # time to change state
                floor_to_service_next = self.elevator.find_floor_to_service_from_top()
                if not floor_to_service_next:
                    self.elevator.transition_to(self.elevator.idle_state)
                    self.elevator.target_floor = None
                else:
                    self.elevator.transition_to(self.elevator.going_up_state)
                    self.elevator.target_floor = floor_to_service_next
            self.elevator.state.serve()
    
        def take_request(self, from_floor: int, direction: Direction):
            super().take_request(from_floor, direction)
            if direction == Direction.down:
                self.elevator.target_floor = min(self.elevator.target_floor, from_floor)
    
    
    class IdleState(ElevatorState):
        def serve(self):
            super().serve()
            time.sleep(1)
            self.elevator.state.serve()
    
        def take_request(self, from_floor: int, direction: Direction):
            super().take_request(from_floor, direction)
            self.elevator.target_floor = from_floor
            if self.elevator.current_floor < self.elevator.target_floor:
                self.elevator.transition_to(self.elevator.going_up_state)
                return
            self.elevator.transition_to(self.elevator.going_down_state)
    ```
- Finally, we have the subject `Elevator` which holds the reference to all the states and can be in any one of them at any given time. It also has the capability to `transition_to` another state which is delegated responsibility of the respective states that the Elevator can be in.
    ```python
    class Elevator:
        current_floor = 0
        target_floor = None
    
        def __init__(self, total_floors: int):
            self.going_up_state = GoingUpState(self)
            self.going_down_state = GoingDownState(self)
            self.idle_state = IdleState(self)
            self.state = self.idle_state
            self.floor_requests = [FloorRequest() for _ in range(total_floors)]
    
        def transition_to(self, state):
            self.state = state
    
        def take_request(self, from_floor: int, direction: Direction):
            self.state.take_request(from_floor, direction)
    
        def serve(self):
            self.state.serve()
    ```

### Implementation
The following is the code for a fully functional model of what we just discussed.

```python
import enum
import logging
import threading
import time
from abc import ABC, abstractmethod

# region Logging
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    handlers=[
        logging.FileHandler("log_file.log"),
    ]
)
logger = logging.getLogger(__name__)
# endregion

# region Enums
class Direction(enum.Enum):
    up = 1
    down = 2
# endregion

# region state abstract class
class ElevatorState(ABC):
    elevator = None

    def __init__(self, elevator_):
        self.elevator = elevator_

    def go_up_one_floor(self):
        time.sleep(1)
        self.elevator.current_floor += 1

    def go_down_one_floor(self):
        time.sleep(1)
        self.elevator.current_floor -= 1

    def set_floor_request_direction(self, floor: int, direction: Direction):
        if direction == Direction.up:
            self.elevator.floor_requests[floor].go_up = True
            return
        self.elevator.floor_requests[floor].go_down = True

    @abstractmethod
    def take_request(self, from_floor: int, direction: Direction):
        self.set_floor_request_direction(from_floor, direction)

    @abstractmethod
    def serve(self):
        logger.info("Current Floor: %s", self.elevator.current_floor)
# endregion

# region Concrete Elevator States
class GoingUpState(ElevatorState):
    def serve(self):
        super().serve()
        self.elevator.floor_requests[self.elevator.current_floor].go_up = False
        # if you've reached the target floor then check whether you change direction or go idle
        if self.elevator.current_floor < self.elevator.target_floor:
            self.go_up_one_floor()
        else:
            # time to change state
            floor_to_service_next = self.elevator.find_floor_to_service_from_bottom()
            if not floor_to_service_next:
                self.elevator.transition_to(self.elevator.idle_state)
                self.elevator.target_floor = None
            else:
                self.elevator.transition_to(self.elevator.going_down_state)
                self.elevator.target_floor = floor_to_service_next
        self.elevator.state.serve()

    def take_request(self, from_floor: int, direction: Direction):
        super().take_request(from_floor, direction)
        if direction == Direction.up:
            self.elevator.target_floor = max(self.elevator.target_floor, from_floor)


class GoingDownState(ElevatorState):
    def serve(self):
        super().serve()
        self.elevator.floor_requests[self.elevator.current_floor].go_down = False
        # if you've reached the target floor then check whether you change direction or go idle
        if self.elevator.current_floor > self.elevator.target_floor:
            self.go_down_one_floor()
        else:
            # time to change state
            floor_to_service_next = self.elevator.find_floor_to_service_from_top()
            if not floor_to_service_next:
                self.elevator.transition_to(self.elevator.idle_state)
                self.elevator.target_floor = None
            else:
                self.elevator.transition_to(self.elevator.going_up_state)
                self.elevator.target_floor = floor_to_service_next
        self.elevator.state.serve()

    def take_request(self, from_floor: int, direction: Direction):
        super().take_request(from_floor, direction)
        if direction == Direction.down:
            self.elevator.target_floor = min(self.elevator.target_floor, from_floor)


class IdleState(ElevatorState):
    def serve(self):
        super().serve()
        time.sleep(1)
        self.elevator.state.serve()

    def take_request(self, from_floor: int, direction: Direction):
        super().take_request(from_floor, direction)
        self.elevator.target_floor = from_floor
        if self.elevator.current_floor < self.elevator.target_floor:
            self.elevator.transition_to(self.elevator.going_up_state)
            return
        self.elevator.transition_to(self.elevator.going_down_state)
# endregion

# a dataclass that represents whether people on a certain floor want to go up or down
class FloorRequest:
    def __init__(self):
        self.go_up = False
        self.go_down = False


class Elevator:
    current_floor = 0
    target_floor = None

    def __init__(self, total_floors: int):
        self.going_up_state = GoingUpState(self)
        self.going_down_state = GoingDownState(self)
        self.idle_state = IdleState(self)
        self.state = self.idle_state
        self.floor_requests = [FloorRequest() for _ in range(total_floors)]

    def find_floor_to_service_from_bottom(self):
        for index, floor in enumerate(self.floor_requests):
            if floor.go_up or floor.go_down:
                return index

    def find_floor_to_service_from_top(self):
        for index, floor in reversed(list(enumerate(self.floor_requests))):
            if floor.go_up or floor.go_down:
                return index

    def transition_to(self, state):
        self.state = state

    def take_request(self, from_floor: int, direction: Direction):
        self.state.take_request(from_floor, direction)

    def serve(self):
        self.state.serve()

    def serve_forever(self):
        threading.Thread(target=self.serve, daemon=True).start()

    def take_requests_forever(self):
        while True:
            floor = int(input("what floor are you at: "))
            direction = input("which direction do you want to go (u/U/d/D): ")
            if direction in {'u', 'U'}:
                direction = Direction.up
            else:
                direction = Direction.down
            self.take_request(floor, direction)


if __name__ == '__main__':
    elevator = Elevator(101)
    elevator.serve_forever()
    elevator.take_requests_forever()
```

To run:
```
python3 elevator.py
```

The program outputs the logs on a separate log file. To view:

```
tail -f log_file.log
```

## Verdict

State Pattern is an amazingly neat way of modelling your FSMs. If you have too many if else statements or switch statements in your logic then you might be able to apply this design pattern to simplify huge methods. But again, its not a silver bullet, use it only when you feel it gives a better structure to your logic.  
It's a little tricky to grasp at first until you implement your own, so go write your own taking this article as your reference.  

**Stay Frosty!**