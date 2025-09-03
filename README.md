# didactic-winner
STATS: List[str] = ["Siła", "Zręczność", "Spostrzegawczość", "Magia", "Charyzma"]
ASSIGN_ORDER_DEFAULT: List[str] = ["Siła", "Spostrzegawczość", "Magia", "Zręczność", "Charyzma"]

# mapowanie broni: (statystyka, ścianki kości obrażeń)
WEAPONS: Dict[str, Tuple[str, int]] = {
    "Miecz krótki": ("Siła", 6),
    "Łuk": ("Spostrzegawczość", 6),
    "Kij magiczny": ("Magia", 4),
}

# -------------------- Funkcje narzędziowe --------------------

def roll_dice(sides: int, rng: Optional[random.Random] = None) -> int:
    """Zwraca rzut kością o zadanej liczbie ścian. Determinizm przez podanie rng."""
    rng = rng or random
    return rng.randint(1, sides)


def get_modifier(value: int) -> int:
    """Modyfikator według tabeli w treści zadania."""
    if value <= 1:
        return 0
    if 2 <= value <= 4:
        return 1
    if 5 <= value <= 7:
        return 2
    if 8 <= value <= 10:
        return 3
    return 4


# -------------------- Generowanie i przypisywanie statystyk --------------------

def generate_stat_rolls(rng: Optional[random.Random] = None) -> List[int]:
    """Generuje listę 5 wartości statystyk wg: k6 + [2,1,0,-1,-2].
    Zwraca listę w tej kolejności modyfikatorów.
    """
    modifiers = [2, 1, 0, -1, -2]
    return [roll_dice(6, rng) + m for m in modifiers]


def auto_assign_stats(rolls: List[int], order: Optional[List[str]] = None) -> Dict[str, int]:
    """Automatyczne przypisanie: sortuj malejąco i przypisz wg kolejności `order`.
    Domyślnie: Siła, Spostrzegawczość, Magia, Zręczność, Charyzma.
    """
    if len(rolls) != 5:
        raise ValueError("Oczekiwano dokładnie 5 wartości rzutów.")
    order = order or ASSIGN_ORDER_DEFAULT
    sorted_rolls = sorted(rolls, reverse=True)
    return {stat: sorted_rolls[i] for i, stat in enumerate(order)}


def assign_stats_from_mapping(rolls: List[int], assignment: Dict[str, int]) -> Dict[str, int]:
    """Waliduje ręczne przypisanie (użyteczne w testach). Każdy rzut musi być użyty raz."""
    if set(assignment.keys()) != set(STATS):
        raise ValueError("Assignment musi zawierać dokładnie te statystyki: " + ", ".join(STATS))
    # sprawdź multiset wartości
    remaining = rolls.copy()
    for stat, val in assignment.items():
        if val not in remaining:
            raise ValueError(f"Wartość {val} nie występuje w dostępnych rzutach {remaining} (stat {stat}).")
        remaining.remove(val)
    return assignment


# -------------------- Logika walki --------------------

def roll_to_hit(weapon: str, stats: Dict[str, int], defense: int, rng: Optional[random.Random] = None) -> Tuple[int, int, bool]:
    """Zwraca (hit_roll, total_hit, success). total_hit = hit_roll + odpowiednia statystyka."""
    if weapon not in WEAPONS:
        raise ValueError(f"Nieznana broń: {weapon}")
    stat_for_weapon, _ = WEAPONS[weapon]
    hit_roll = roll_dice(20, rng)
    total = hit_roll + stats[stat_for_weapon]
    return hit_roll, total, total >= defense


def roll_damage(weapon: str, stats: Dict[str, int], armor: int, rng: Optional[random.Random] = None) -> Tuple[int, int, int, int]:
    """Zwraca (rzucona_kość, mod, obrażenia_surowe, obrażenia_końcowe_po_pancerzu)."""
    if weapon not in WEAPONS:
        raise ValueError(f"Nieznana broń: {weapon}")
    stat_for_weapon, dmg_die = WEAPONS[weapon]
    dmg_roll = roll_dice(dmg_die, rng)
    mod = get_modifier(stats[stat_for_weapon])
    raw = dmg_roll + mod
    final = max(0, raw - armor)
    return dmg_roll, mod, raw, final


def simulate_attack(
    weapon: str,
    defense: int,
    armor: int,
    stats: Dict[str, int],
    rng: Optional[random.Random] = None,
) -> Dict[str, int]:
    """Symuluje pełny atak. Zwraca słownik z danymi przebiegu ataku."""
    hit_roll, total_hit, success = roll_to_hit(weapon, stats, defense, rng)
    result = {
        "hit_roll": hit_roll,
        "total_hit": total_hit,
        "hit_success": int(success),  # 1/0 do łatwiejszego logowania
    }
    if success:
        dr, mod, raw, fin = roll_damage(weapon, stats, armor, rng)
        result.update({
            "dmg_roll": dr,
            "dmg_mod": mod,
            "dmg_raw": raw,
            "dmg_final": fin,
        })
    else:
        result.update({
            "dmg_roll": 0,
            "dmg_mod": 0,
            "dmg_raw": 0,
            "dmg_final": 0,
        })
    return result


