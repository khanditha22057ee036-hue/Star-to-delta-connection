"""
Star-Delta (Y-Δ) Connection Calculator
--------------------------------------
A menu-driven program for engineering students to analyse a three-phase
system with a STAR-connected source feeding a DELTA-connected load, a
star-delta transformer, and star-to-delta impedance conversion.

Balanced star-delta system (convert the load to an equivalent star):
    Source phase voltage      Vph = VL / sqrt(3)
    Equivalent star load      Z_Y = Z_delta / 3
    Line current              IL = Vph / |Zline + Z_delta/3|
    Load phase current        Iph = IL / sqrt(3)
    Load phase voltage        V_load = Iph * |Z_delta|  (= load line voltage)
    Load power                P = 3 * Iph^2 * R_delta
    Source phase current      = IL  (star source)

Unbalanced star-delta system (phasor analysis):
    1. Convert the load delta to an equivalent star:
           Za = Zab*Zca / (Zab+Zbc+Zca)    Zb = Zab*Zbc / (Zab+Zbc+Zca)
           Zc = Zbc*Zca / (Zab+Zbc+Zca)
    2. Source phase voltages Van, Vbn, Vcn (ABC sequence, Van as reference).
    3. Neutral shift by Millman's theorem, then the line currents.
    4. Load phase currents: Iab = V_AB(load) / Zab, etc.

Star-delta transformer:
    Secondary line voltage    VL2 = VL1 / (sqrt(3) * a)        (a = N1 / N2)
    Primary line current      IL1 = S / (sqrt(3) * VL1)
    Secondary winding current = IL2 / sqrt(3)
    Phase shift of 30 degrees between primary and secondary line voltages

Star to delta impedance conversion:
    Z_ab = (Za*Zb + Zb*Zc + Zc*Za) / Zc
    Z_bc = (Za*Zb + Zb*Zc + Zc*Za) / Za
    Z_ca = (Za*Zb + Zb*Zc + Zc*Za) / Zb
    Balanced case: Z_delta = 3 * Z_star
"""

import cmath
import math

SQRT3 = math.sqrt(3)


# ---------------------------------------------------------------- input helpers
def get_positive_float(prompt):
    """Keep asking until the user enters a valid positive number."""
    while True:
        try:
            value = float(input(prompt))
            if value <= 0:
                print("  Please enter a value greater than zero.")
                continue
            return value
        except ValueError:
            print("  Invalid input. Please enter a number.")


def get_nonneg_float(prompt):
    """Ask for a number that may be zero but not negative."""
    while True:
        try:
            value = float(input(prompt))
            if value < 0:
                print("  Please enter zero or a positive value.")
                continue
            return value
        except ValueError:
            print("  Invalid input. Please enter a number.")


def get_float(prompt):
    """Ask for any number (zero and negative values allowed)."""
    while True:
        try:
            return float(input(prompt))
        except ValueError:
            print("  Invalid input. Please enter a number.")


def get_impedance(label, who="Load"):
    """Ask for resistance and reactance of one phase; return complex Z.
    Reactance: positive for inductive, negative for capacitive."""
    r = get_positive_float(f"  {who} {label}: resistance R (ohm): ")
    x = get_float(f"  {who} {label}: reactance X (ohm, - for capacitive): ")
    return complex(r, x)


def get_line_impedance():
    """Ask for the impedance of each line conductor (may be zero)."""
    print("  Line (cable) impedance per phase - enter 0 and 0 if ignored:")
    r = get_nonneg_float("  Line resistance (ohm): ")
    x = get_nonneg_float("  Line reactance (ohm): ")
    return complex(r, x)


# ------------------------------------------------------------------ core maths
def delta_to_star(zab, zbc, zca):
    """Equivalent star impedances (Za, Zb, Zc) of any (balanced or not) delta."""
    total = zab + zbc + zca
    return zab * zca / total, zab * zbc / total, zbc * zca / total


def star_to_delta(za, zb, zc):
    """Equivalent delta impedances (Zab, Zbc, Zca) of any (balanced or not) star."""
    s = za * zb + zb * zc + zc * za
    return s / zc, s / za, s / zb


def neutral_shift(voltages, impedances):
    """Millman's theorem: voltage between load neutral and source neutral."""
    admittances = [1 / z for z in impedances]
    numerator = sum(v * y for v, y in zip(voltages, admittances))
    return numerator / sum(admittances)


# ------------------------------------------------------------------- display
def show_polar(name, value, unit):
    """Print a complex quantity in polar form."""
    mag, ang = cmath.polar(value)
    print(f"  {name} = {mag:.3f} < {math.degrees(ang):.2f} deg {unit}")


def show_rect(name, z):
    """Print an impedance in rectangular form."""
    sign = "+" if z.imag >= 0 else "-"
    print(f"  {name} = {z.real:.3f} {sign} j{abs(z.imag):.3f} ohm")


def balanced_star_delta():
    v_l = get_positive_float("Source (star) line voltage VL (V): ")
    z_line = get_line_impedance()
    z_delta = get_impedance("(each delta phase)")

    v_ph = v_l / SQRT3
    z_total = z_line + z_delta / 3
    i_l = v_ph / abs(z_total)
    i_ph = i_l / SQRT3
    v_load = i_ph * abs(z_delta)
    pf = z_delta.real / abs(z_delta)
    s_load = 3 * i_ph ** 2 * abs(z_delta)
    p_load = 3 * i_ph ** 2 * z_delta.real
    q_load = 3 * i_ph ** 2 * z_delta.imag
    p_loss = 3 * i_l ** 2 * z_line.real
    drop = (v_l - v_load) / v_l * 100
    eff = p_load / (p_load + p_loss) * 100

    print("\n  ----- Balanced Star-Delta System -----")
    print(f"  Source phase voltage    = {v_ph:.3f} V")
    print(f"  Line current IL         = {i_l:.4f} A (= source phase current)")
    print(f"  Load phase current Iph  = {i_ph:.4f} A")
    print(f"  Load phase/line voltage = {v_load:.3f} V")
    print(f"  Voltage drop in lines   = {drop:.2f} %")
    print(f"  Load power factor       = {pf:.4f} {'lagging' if z_delta.imag > 0 else 'leading' if z_delta.imag < 0 else '(unity)'}")
    print(f"  Load apparent power S   = {s_load:,.2f} VA")
    print(f"  Load active power P     = {p_load:,.2f} W")
    print(f"  Load reactive power Q   = {q_load:,.2f} VAR")
    print(f"  Line power loss         = {p_loss:,.2f} W")
    print(f"  Transmission efficiency = {eff:.2f} %")


def unbalanced_star_delta():
    v_l = get_positive_float("Source (star) line voltage VL (V): ")
    z_line = get_line_impedance()
    print("\n  Enter the impedance of each delta phase of the load:")
    zab, zbc, zca = get_impedance("AB"), get_impedance("BC"), get_impedance("CA")

    # Star source phase voltages (ABC sequence, Van as reference)
    v_ph = v_l / SQRT3
    van = cmath.rect(v_ph, 0)
    vbn = cmath.rect(v_ph, math.radians(-120))
    vcn = cmath.rect(v_ph, math.radians(120))

    za, zb, zc = delta_to_star(zab, zbc, zca)
    totals = [z_line + za, z_line + zb, z_line + zc]

    v_nn = neutral_shift([van, vbn, vcn], totals)
    ia, ib, ic = [(v - v_nn) / zt for v, zt in zip([van, vbn, vcn], totals)]

    # Load terminal line voltages and delta phase currents
    v_ab_load = ia * za - ib * zb
    v_bc_load = ib * zb - ic * zc
    v_ca_load = ic * zc - ia * za
    iab, ibc, ica = v_ab_load / zab, v_bc_load / zbc, v_ca_load / zca

    p_total = (abs(iab) ** 2 * zab.real + abs(ibc) ** 2 * zbc.real + abs(ica) ** 2 * zca.real)
    p_loss = (abs(ia) ** 2 + abs(ib) ** 2 + abs(ic) ** 2) * z_line.real

    print("\n  ----- Unbalanced Star-Delta System -----")
    show_polar("Line current Ia ", ia, "A")
    show_polar("Line current Ib ", ib, "A")
    show_polar("Line current Ic ", ic, "A")
    show_polar("Load phase current Iab", iab, "A")
    show_polar("Load phase current Ibc", ibc, "A")
    show_polar("Load phase current Ica", ica, "A")
    show_polar("Load voltage Vab", v_ab_load, "V")
    show_polar("Load voltage Vbc", v_bc_load, "V")
    show_polar("Load voltage Vca", v_ca_load, "V")
    print(f"  Total load active power P = {p_total:,.2f} W")
    print(f"  Total line loss           = {p_loss:,.2f} W")
    print(f"  Check: Ia + Ib + Ic       = {abs(ia + ib + ic):.6f} A (must be zero)")


def star_delta_transformer():
    v1 = get_positive_float("Primary (star) line voltage VL1 (V): ")
    turns1 = get_positive_float("Primary turns per phase N1: ")
    turns2 = get_positive_float("Secondary turns per phase N2: ")
    kva = get_positive_float("Total three-phase rating (kVA): ")

    a = turns1 / turns2
    v1_ph = v1 / SQRT3
    v2 = v1_ph / a
    il1 = kva * 1000 / (SQRT3 * v1)
    il2 = kva * 1000 / (SQRT3 * v2)

    print("\n  ----- Star-Delta Transformer -----")
    print(f"  Turns ratio a = N1/N2        = {a:.4f}")
    print(f"  Primary phase voltage        = {v1_ph:.3f} V")
    print(f"  Secondary phase voltage      = {v2:.3f} V (= line voltage)")
    print(f"  Line voltage ratio           = {v2 / v1:.4f} (= 1/(sqrt(3)*a))")
    print(f"  Primary line/phase current   = {il1:.3f} A")
    print(f"  Secondary line current       = {il2:.3f} A")
    print(f"  Secondary winding current    = {il2 / SQRT3:.3f} A")
    print(f"  Rating of each transformer   = {kva / 3:.3f} kVA")
    print("  Phase shift of 30 degrees between primary and secondary line voltages.")
    print("  The delta winding traps third-harmonic currents and helps stabilise voltages.")


def impedance_conversion():
    print("  1. Balanced star (all three phases equal)")
    print("  2. Unbalanced star (three different impedances)")
    while True:
        pick = input("  Enter 1 or 2: ").strip()
        if pick in ("1", "2"):
            break
        print("  Please enter 1 or 2.")

    print("\n  ----- Equivalent Delta Impedances -----")
    if pick == "1":
        z = get_impedance("(balanced)", who="Star")
        show_rect("Z_delta (each phase)", 3 * z)
    else:
        za = get_impedance("A", who="Star")
        zb = get_impedance("B", who="Star")
        zc = get_impedance("C", who="Star")
        zab, zbc, zca = star_to_delta(za, zb, zc)
        print()
        show_rect("Z_ab", zab)
        show_rect("Z_bc", zbc)
        show_rect("Z_ca", zca)


def menu():
    print("\n" + "=" * 54)
    print("        STAR-DELTA CONNECTION CALCULATOR")
    print("=" * 54)
    print(" 1. Balanced star-delta system (with line impedance)")
    print(" 2. Unbalanced star-delta system (phasor analysis)")
    print(" 3. Star-delta three-phase transformer")
    print(" 4. Star to delta impedance conversion")
    print(" 0. Exit")
    print("-" * 54)


def main():
    while True:
        menu()
        choice = input("Enter your choice: ").strip()

        if choice == "1":
            balanced_star_delta()
        elif choice == "2":
            unbalanced_star_delta()
        elif choice == "3":
            star_delta_transformer()
        elif choice == "4":
            impedance_conversion()
        elif choice == "0":
            print("\nThank you for using the calculator. Goodbye!")
            break
        else:
            print("  Invalid choice. Please select from the menu.")


if __name__ == "__main__":
    main()
