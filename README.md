[JARVIS.py](https://github.com/user-attachments/files/26884502/JARVIS.py)
[executor.py](https://github.com/user-attachments/files/26884450/executor.py)[brain.py](https://github.com/user-attachments/files/26884453/brain.py)
[__init__.py](https://github.com/user-attachments/files/26884452/__init__.py)
[voice.py](https://github.com/user-attachments/files/26884451/voice.py)
[settings.py](https://github.com/user-attachments/files/26884470/settings.py)


            logger.error("Google Speech API error: %s", e)

    # ── Process command ────────────────────────────────────────────────────────

    def process_command(self, command: str) -> None:
        if not command or not isinstance(command, str):
            return
        command = command.strip()
        if not command:
            return

        try:
            with self._lock:
                print(f'\nYou said: "{command}"')
                logger.info("Processing command: %s", command)

                self.display.show_thinking()
                response, actions = self.brain.process(command)

                # Check if brain wants JARVIS to shut down
                if actions.get("shutdown"):
                    self.display.show_response(response)
                    self.voice.speak(response)
                    self.shutdown()
                    return

                # Execute system actions
                if actions:
                    action_result = self.executor.run_actions(actions)
                    if action_result:
                        print(f"[ACTION] {action_result}")
                        logger.info("Actions executed: %s", action_result)

                self.display.show_response(response)
                self.voice.speak(response)
                logger.info("Response delivered")

        except Exception as e:
            logger.error("Command processing error: %s", e)
            self.voice.speak("An error occurred processing your command, sir.")

    # ── Web server ─────────────────────────────────────────────────────────────

    def _start_web_server(self) -> None:
        try:
            from phone_server.server import start_phone_server
            t = threading.Thread(target=start_phone_server, kwargs={"jarvis": self}, daemon=True)
            t.start()
            logger.info("Web control server started on port 5000")
        except ImportError:
            logger.warning("Phone server module not found, skipping web UI")
            print("[!] Phone server module not found, skipping web UI\n")
        except Exception as e:
            logger.warning("Web server startup failed: %s", e)

    # ── Run ────────────────────────────────────────────────────────────────────

    def run(self) -> None:
        try:
            self.boot_sequence()
            print("\n[*] Say 'Jarvis' to activate  |  Ctrl+C to shutdown\n")
            print("=" * 55 + "\n")
            self._start_web_server()
            self.listen_for_wake_word()
        except KeyboardInterrupt:
            logger.info("Keyboard interrupt received")
            self.shutdown()
        except Exception as e:
            logger.error("Fatal error in main loop: %s", e)
            self.shutdown()

    # ── Shutdown ───────────────────────────────────────────────────────────────

    def shutdown(self) -> None:
        try:
            logger.info("Initiating shutdown sequence")
            self.running = False
            self.display.show_shutdown()
            if hasattr(self.voice, "cleanup"):
                self.voice.cleanup()
            logger.info("Shutdown complete")
            sys.exit(0)
        except Exception as e:
            logger.error("Shutdown error: %s", e)
            sys.exit(1)


if __name__ == "__main__":
    try:
        jarvis = Jarvis()
        jarvis.run()
    except KeyboardInterrupt:
        sys.exit(0)
    except Exception as e:
        logger.critical("Critical error: %s", e)
        sys.exit(1)

