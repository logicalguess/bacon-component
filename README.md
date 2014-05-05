## Using streams to define event-driven components

![ ](screen1.png) ![ ](screen2.png)
![ ](screen3.png) ![ ](screen4.png)

```javascript

        function wireAjaxOnChange(input, svcInfo, init) {
            var request = input.changes().filter(nonEmpty).skipDuplicates().throttle(300)
                    .map(svcInfo);

            var response = request.ajax();

            return {
                requestEntered: input.map(nonEmpty),
                responsePending: request.awaiting(response),
                responseValue: response.toProperty(init)
            }
        }

        function wireAjaxOnEvent(eventStream, svcTemplate) {
            var request = Bacon.combineTemplate(svcTemplate).sampledBy(eventStream);
            var response = request.ajax();

            return {
                requestEntered: request.map(true).toProperty(false),
                responsePending: request.awaiting(response),
                responseStream: response
            }
        }

        function createComponent() {
            var model = new Bacon.Model({});
            var lenses = {};
            lenses["username"] = model.lens("username");
            lenses["fullname"] = model.lens("fullname");

            var streams = {};
            streams["username_availability"] = new Bacon.Bus();
            streams["register"] = new Bacon.Bus();

            streams["username_availability"].plug(lenses["username"]);

            var userNameWire = wireAjaxOnChange(streams["username_availability"].toProperty(), function (user) {
                return { url: "/usernameavailable/" + user };
            }, true);

            userNameWire.requestEntered.onValue(function () {
                lenses["fullname"].set("");
            });

            // registration
            var registrationWire = wireAjaxOnEvent(streams["register"], {
                type: "post",
                url: "/register",
                contentType: "application/json",
                data: JSON.stringify(model.get())
            });

            registrationWire.responseStream.onValue(function () {
                model.set({username: "", fullname: ""});
            });

            return {
                onStateChange: model.onValue.bind(model),
                lenses: lenses,
                bind: function(name, stream) {
                    lenses[name].bind(stream);
                },
                plug: function(name, stream) {
                    streams[name].plug(stream);
                },
                usernameEntered: userNameWire.requestEntered,
                usernameAvailable: userNameWire.responseValue,
                availabilityPending: userNameWire.responsePending,
                registrationPending: registrationWire.responsePending,
                registrationSent: registrationWire.requestEntered,
                registrationResponse: registrationWire.responseStream
            }
        }

        function show(x) {
            console.log(x);
        }

        function nonEmpty(x) {
            return x && x.length > 0;
        }

        function setVisibility(element, visible) {
            element.toggle(visible);
        }

        function setEnabled(element, enabled) {
            element.attr("disabled", !enabled);
        }

        function createView(component) {

            // UI elements
            var elems = {};
            elems["username"] = $("#username input");
            elems["fullname"] = $("#fullname input");
            elems["register"] = $("#register button");
            elems["unavailability_label"] = $("#username-unavailable");
            elems["username_availability_spinner"] = $("#username .ajax");
            elems["register_spinner"] = $("#register .ajax");

            // bindings
            component.plug("register", elems["register"].asEventStream("click").doAction(".preventDefault"));

            component.bind("username", Bacon.$.textFieldValue(elems["username"]));
            component.bind("fullname", Bacon.$.textFieldValue(elems["fullname"]));
            component.onStateChange(function (m) {
                $("#result").text("");
                console.log("model", m);
            });

            // visual effects
            component.usernameAvailable.not().and(component.availabilityPending.not()).onValue(setVisibility,
                    elems["unavailability_label"]);
            component.availabilityPending.onValue(setVisibility, elems["username_availability_spinner"]);


            var fullnameEntered = component.lenses["fullname"].map(nonEmpty);

            var fullnameEnabled = component.usernameEntered.and(component.usernameAvailable)
                    .and(component.availabilityPending.not());

            var registerButtonEnabled = component.usernameEntered.and(fullnameEntered).and(component.usernameAvailable)
                    .and(component.availabilityPending.not()).and(component.registrationSent.not());

            fullnameEnabled.onValue(setEnabled, elems["fullname"]);
            registerButtonEnabled.onValue(setEnabled, elems["register"]);
            component.registrationPending.onValue(setVisibility, elems["register_spinner"]);
            component.registrationResponse.onValue(function () {
                $("#result").text("Thanks dude!");
            })
        }

        $(function () {
            createView(createComponent());
        })
```