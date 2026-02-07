Below is the complete list of tools available to me in this environment, with their exact names, purposes, and function signatures as defined. This is a technical inventory, not a usage guide.


---

🧠 Context & Memory

personal_context.search

Retrieves user-specific historical context (preferences, prior decisions, ongoing work).

search({
  query: string
})

bio.update

Persistently stores or deletes long-term user memory.

update(text: string)


---

🌐 Web & External Information

web.run

Searches or opens live web pages for current, verifiable information.

run({
  search_query?: [{ q: string }],
  open?: [{ ref_id: string }]
})


---

📁 File & Document Search

file_search.msearch

Searches connected/internal knowledge sources and uploaded files.

msearch({
  queries: string[],
  source_filter: string[],
  file_type_filter?: string[],
  intent?: string,
  time_frame_filter?: {
    start_date?: string,
    end_date?: string
  }
})

file_search.mclick

Opens and expands previously found file/document results.

mclick({
  pointers: string[],
  start_date?: string,
  end_date?: string
})


---

🧮 Python Execution

python.exec

Runs private Python code for internal reasoning only (no user-visible output).

exec(code: string)

python_user_visible.exec

Runs user-visible Python code (tables, plots, files).

exec(code: string)


---

🖼️ Image Generation & Editing

image_gen.text2im

Generates or edits images based on a prompt or reference image.

text2im({
  prompt?: string,
  size?: string,
  n?: number,
  transparent_background?: boolean,
  is_style_transfer?: boolean,
  referenced_image_ids?: string[]
})


---

🧩 UI Widgets & Utilities

genui.search

Finds appropriate interactive widgets (weather, currency, calculator, time).

search({
  query: string
})

genui.run

Renders a previously discovered widget.

run()


---

🗓️ Automation & Scheduling

automations.create

Creates reminders, scheduled searches, or conditional checks.

create({
  title: string,
  prompt: string,
  schedule?: string,
  dtstart_offset_json?: string
})

automations.update

Updates or enables/disables an automation.

update({
  jawbone_id: string,
  schedule?: string,
  dtstart_offset_json?: string,
  prompt?: string,
  title?: string,
  is_enabled?: boolean
})

automations.list

Lists all active automations.

list()


---

📬 Email (Read-Only)

gmail.search_email_ids

Searches Gmail messages.

search_email_ids({
  query?: string,
  tags?: string[],
  max_results?: number,
  next_page_token?: string
})

gmail.batch_read_email

Reads one or more Gmail messages.

batch_read_email({
  message_ids: string[]
})


---

📆 Google Calendar (Read-Only)

gcal.search_events

Searches calendar events.

search_events({
  time_min?: string,
  time_max?: string,
  timezone_str?: string,
  max_results?: number,
  query?: string,
  calendar_id?: string,
  next_page_token?: string
})

gcal.read_event

Reads a specific calendar event.

read_event({
  event_id: string,
  calendar_id?: string
})


---

👥 Google Contacts (Read-Only)

gcontacts.search_contacts

Searches contacts.

search_contacts({
  query: string,
  max_results?: number
})


---

🧱 Canvas / Document Workspace

canmore.create_textdoc

Creates a new canvas document or code file.

create_textdoc({
  name: string,
  type: "document" | "code/*",
  content: string
})

canmore.update_textdoc

Edits an existing canvas document.

update_textdoc({
  updates: [{
    pattern: string,
    multiple?: boolean,
    replacement: string
  }]
})

canmore.comment_textdoc

Adds inline comments to a canvas document.

comment_textdoc({
  comments: [{
    pattern: string,
    comment: string
  }]
})


---

🛠️ System / Containers

container.exec

Executes shell commands in a container.

exec({
  cmd: string[],
  session_name?: string,
  workdir?: string,
  timeout?: number,
  env?: object,
  user?: string
})

container.feed_chars

Sends input to a running container session.

feed_chars({
  session_name: string,
  chars: string,
  yield_time_ms?: number
})

container.open_image

Reads an image file from the container filesystem.

open_image({
  path: string
})

container.download

Downloads a file into the container.

download({
  url: string,
  filepath: string
})


---

🧰 API Resource Explorer

api_tool.list_resources

Lists available API-style resources and tools.

list_resources({
  path?: string,
  cursor?: string,
  only_tools?: boolean,
  refetch_tools?: boolean
})

api_tool.call_tool

Invokes a discovered API resource.

call_tool({
  path: string,
  args: object
})


---

🎛️ User Settings

user_settings.get_user_settings

Retrieves current UI and personality settings.

get_user_settings()

user_settings.set_setting

Updates accent color, appearance, or personality.

set_setting({
  setting_name: "accent_color" | "appearance" | "personality",
  setting_value: string
})


---

📦 Artifact Handoff

artifact_handoff.prepare_artifact_generation

Initialises spreadsheet or slide generation.

prepare_artifact_generation()


---

Closing note

This list reflects capability, not permission. Tools are invoked only when appropriate, safe, and necessary—and many are read-only by design.