Here's the complete end-to-end flow from Enter key to model API:              
                                                                            
[TUI layer]                                                                 
Enter key                                                                   
    → handle_key_event()                      chatwidget/interaction.rs
    → handle_composer_input_result()          chatwidget/input_flow.rs
    → submit_user_message()                   chatwidget/input_submission.rs
    → AppCommand::user_turn(items, model, …)
    → submit_op()  →  AppEvent::CodexOp(op)   chatwidget.rs

[App event loop]
    → handle_event()                          app/event_dispatch.rs
    → submit_active_thread_op()
    → try_submit_active_thread_op_via_app_server()  app/thread_routing.rs
    → app_server.turn_start(items, model, …)

[App-server layer]
    → ClientRequest::TurnStart               app_server_session.rs
    → turn_start_inner()                     app-server/turn_processor.rs
    → Op::UserInput { items }
    → thread.submit_user_input()

[Core agent layer]
    → tx_sub.send(Submission)                core/session/mod.rs
    → submission_loop() receives it          core/session/handlers.rs
    → user_input_or_turn()
    → sess.spawn_task(RegularTask)

[Task / turn layer]
    → tokio::spawn(task.run())               core/tasks/mod.rs
    → RegularTask::run()                     core/tasks/regular.rs
    → run_turn()                             core/session/turn.rs
    → client_session.stream(prompt, model)   ← HTTP request to OpenAI API
    → streams back tokens → emits events