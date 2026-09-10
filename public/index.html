const eventGrid = document.getElementById("event-grid");
const eventsStatus = document.getElementById("events-status");
const filtersEl = document.getElementById("filters");
const modal = document.getElementById("register-modal");
const form = document.getElementById("register-form");
const eventSelect = document.getElementById("event-select");
const formError = document.getElementById("form-error");
const formView = document.getElementById("modal-form-view");
const successView = document.getElementById("modal-success-view");
const successDetail = document.getElementById("success-detail");

let events = [];
let activeCategory = "All";
const originalSpots = {};

function setStatus(message) {
  eventsStatus.hidden = !message;
  eventsStatus.textContent = message || "";
}

function spotsLabel(spots) {
  if (spots <= 0) {
    return "No spots left";
  }

  return spots === 1 ? "1 spot left" : `${spots} spots left`;
}

function spotsWidth(event) {
  const total = originalSpots[event.id] || event.spots || 1;
  return `${Math.max(0, Math.min(100, (event.spots / total) * 100))}%`;
}

function renderFilters() {
  const categories = ["All", ...new Set(events.map((event) => event.category))];
  filtersEl.innerHTML = "";

  categories.forEach((category) => {
    const button = document.createElement("button");
    button.type = "button";
    button.className = "filter-btn" + (category === activeCategory ? " is-active" : "");
    button.textContent = category;
    button.addEventListener("click", () => {
      activeCategory = category;
      renderFilters();
      renderEvents();
    });
    filtersEl.appendChild(button);
  });
}

function renderEvents() {
  const visibleEvents =
    activeCategory === "All"
      ? events
      : events.filter((event) => event.category === activeCategory);

  eventGrid.innerHTML = "";

  visibleEvents.forEach((event) => {
    const card = document.createElement("article");
    card.className = "event-card";
    card.innerHTML = `
      <span class="category">${event.category}</span>
      <h3>${event.name}</h3>
      <div class="meta">
        <span>${event.date}</span>
        <span>${event.location}</span>
      </div>
      <p>${event.description}</p>
      <div class="spots">
        <span>${spotsLabel(event.spots)}</span>
      </div>
      <div class="spots-bar" aria-hidden="true"><span style="width: ${spotsWidth(event)}"></span></div>
      <button class="card-btn" type="button" ${event.spots <= 0 ? "disabled" : ""}>
        ${event.spots <= 0 ? "Event full" : "Register"}
      </button>
    `;

    const button = card.querySelector(".card-btn");
    button.addEventListener("click", () => openModal(event.id));
    eventGrid.appendChild(card);
  });
}

function fillEventSelect(selectedId) {
  eventSelect.innerHTML = "";

  events.forEach((event) => {
    const option = document.createElement("option");
    option.value = String(event.id);
    option.textContent = event.spots > 0 ? event.name : `${event.name} (full)`;
    option.disabled = event.spots <= 0;
    option.selected = event.id === selectedId;
    eventSelect.appendChild(option);
  });
}

function openModal(eventId) {
  const selectedId = eventId || events.find((event) => event.spots > 0)?.id;
  form.reset();
  formError.hidden = true;
  formView.hidden = false;
  successView.hidden = true;
  fillEventSelect(selectedId);
  modal.hidden = false;
  form.elements.name.focus();
}

function closeModal() {
  modal.hidden = true;
}

async function loadEvents() {
  setStatus("Loading events...");

  try {
    const response = await fetch("/api/events");
    if (!response.ok) {
      throw new Error("Could not load events");
    }

    events = await response.json();
    events.forEach((event) => {
      if (originalSpots[event.id] === undefined) {
        originalSpots[event.id] = event.spots;
      }
    });

    setStatus("");
    renderFilters();
    renderEvents();
  } catch (error) {
    setStatus("Events could not be loaded. Please refresh the page.");
  }
}

form.addEventListener("submit", async (event) => {
  event.preventDefault();
  formError.hidden = true;

  const payload = {
    name: form.elements.name.value.trim(),
    email: form.elements.email.value.trim(),
    eventId: Number(form.elements.eventId.value)
  };

  try {
    const response = await fetch("/api/register", {
      method: "POST",
      headers: {
        "Content-Type": "application/json"
      },
      body: JSON.stringify(payload)
    });

    const data = await response.json();

    if (!response.ok) {
      formError.hidden = false;
      formError.textContent = data.error || "Registration failed.";
      return;
    }

    successDetail.textContent = data.registration
      ? `${data.registration.name} is on the list for ${data.registration.eventName}.`
      : "";
    formView.hidden = true;
    successView.hidden = false;
    await loadEvents();
  } catch (error) {
    formError.hidden = false;
    formError.textContent = "Could not reach the server. Please try again.";
  }
});

document.getElementById("header-register").addEventListener("click", () => openModal());

document.querySelectorAll("[data-close-modal]").forEach((element) => {
  element.addEventListener("click", closeModal);
});

document.addEventListener("keydown", (event) => {
  if (event.key === "Escape" && !modal.hidden) {
    closeModal();
  }
});

loadEvents();
