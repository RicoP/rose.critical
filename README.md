# rose.critical
A lib for savely accessing critical sections for multithreading

## How to use

```C++
struct InputState {
  std::vector<XINPUT> inputs;
};

critical<InputState> g_critical_input_state;

void input_thread() {
  while(g_critical_input_state.alive()) {
    XINPUT state;
    XINPUT_PULL(&state);
    {
      if(auto cx = g_critical_input_state.get_wait(), cx.ok()) { //cx.ok() is always returning true, because get_wait is spinning until resource is free
        cx->inputs.push_back(state);
      } //here getter will be desolved and atomic thread state will be set to idle
    }
    thread_wait(1_ms);
  }
}

std::vector<XINPUT> g_game_inputs;
void game_thread() {
  for(;;) {
    {
      if(auto cx = g_critical_input_state.get(), cx.ok()) { //here cx.ok() may return false when resource was blocked
        g_game_inputs = cx->inputs;
        cx->inputs.clear();
      } //after here g_critical_input_state is again free to grab
    }

    if(g_game_inputs.any([](auto & i){ return i.back_pressed; })) {
      g_critical_input_state.kill(); //set resource state to dead. Trying to get it will always return false.
    }
  }
}
```
