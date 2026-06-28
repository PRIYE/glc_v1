---
description: "Task list for LINE Channel Adapter implementation"
---

# Tasks: LINE Channel Adapter

**Input**: Design documents from `/specs/001-line-channel-adapter/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/adapter-interface.md

**Organization**: Tasks are grouped by user story to enable independent implementation and testing of each story.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (e.g., US1, US2, US3)
- Include exact file paths in descriptions

## Path Conventions

- **Single project**: `glc/channels/catalogue/line/`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Project initialization and basic structure

- [X] T001 Create Pydantic schemas (`LineWebhookSource`, `LineWebhookMessage`, `LineWebhookEvent`, `LineWebhookBody`) in `glc/channels/catalogue/line/schemas.py`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core infrastructure that MUST be complete before ANY user story can be implemented

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [X] T002 Implement `ReplyTokenStore` logic (`__init__` for `_token_store`, `_stash_token`, `_consume_token`) in `glc/channels/catalogue/line/adapter.py`

**Checkpoint**: Foundation ready - user story implementation can now begin

---

## Phase 3: User Story 1 - Inbound Message Parsing & Trust Classification (Priority: P1) 🎯 MVP

**Goal**: Parse incoming LINE webhooks, determine trust level, and return a valid `ChannelMessage`.

**Independent Test**: `pytest tests/channels/test_line.py -k "test_on_message"`

### Implementation for User Story 1

- [X] T003 [US1] Implement `on_message` parsing, trust classification, and `ChannelMessage` construction in `glc/channels/catalogue/line/adapter.py`

**Checkpoint**: At this point, User Story 1 should be fully functional and testable independently

---

## Phase 4: User Story 2 - Reply-Token-First Outbound (Priority: P1)

**Goal**: Use the LINE Reply API when an in-flight reply token exists, and fall back to the Push API when no valid token is available.

**Independent Test**: `pytest tests/channels/test_line.py -k "test_send_emits_valid_wire_payload or test_channel_specific_behaviour_reply_token_then_push"`

### Implementation for User Story 2

- [X] T004 [US2] Implement `send` method to use reply token or fallback to push in `glc/channels/catalogue/line/adapter.py`

**Checkpoint**: At this point, User Stories 1 AND 2 should both work independently

---

## Phase 5: User Story 3 - Resilience & Error Propagation (Priority: P2)

**Goal**: Handle network disruptions gracefully and propagate rate-limit errors.

**Independent Test**: `pytest tests/channels/test_line.py -k "test_disconnect_is_handled or test_rate_limit_propagates_429"`

### Implementation for User Story 3

- [X] T005 [US3] Add disconnect handling to `on_message` and rate limit propagation to `send` in `glc/channels/catalogue/line/adapter.py`

**Checkpoint**: All user stories should now be independently functional

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Improvements that affect multiple user stories

- [X] T006 Run test suite to verify all 7 tests in `tests/channels/test_line.py` pass
- [X] T007 Run `ruff` and `mypy` on `glc/channels/catalogue/line/` to ensure code quality

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies - can start immediately
- **Foundational (Phase 2)**: Depends on Setup completion - BLOCKS all user stories
- **User Stories (Phase 3+)**: All depend on Foundational phase completion
  - User stories can then proceed sequentially in priority order (P1 → P2)
- **Polish (Final Phase)**: Depends on all desired user stories being complete

### User Story Dependencies

- **User Story 1 (P1)**: Can start after Foundational (Phase 2) - No dependencies on other stories
- **User Story 2 (P1)**: Can start after Foundational (Phase 2) - Integrates with US1's token stashing
- **User Story 3 (P2)**: Can start after US1 and US2 are complete

### Within Each User Story

- Core implementation before integration
- Story complete before moving to next priority

### Parallel Opportunities

- T001 and T002 can technically be started in parallel as they touch different files (`schemas.py` vs `adapter.py`).

---

## Implementation Strategy

### Incremental Delivery

1. Complete Setup + Foundational → Foundation ready
2. Add User Story 1 → Test independently
3. Add User Story 2 → Test independently
4. Add User Story 3 → Test independently
5. Each story adds value without breaking previous stories
