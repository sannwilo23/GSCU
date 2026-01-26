# 全球亚文化俱乐部联盟 技术文档2：联盟内部编号（GPC）的实现（Python、C++、Haskell，含测试用例）

Python：

```python
from __future__ import annotations
import math
from typing import Tuple, Union

class GSCUIdentifier:
    """
    Represents a unique identifier within the Global Subculture Clubs Union (GSCU).

    This class provides a robust, immutable object for handling GSCU's hierarchical
    identification codes (Club, Team, Stronghold, Researcher). It can parse from
    and convert to string, integer, and bytes representations.

    Attributes:
        club (str): The alphabetic codename of the club (e.g., 'LFN').
        team (str | None): The numeric ID of the team.
        stronghold (str | None): The numeric ID of the stronghold.
        researcher (str | None): The numeric ID of the researcher.
        level (str): The level of the identifier ('club', 'team', 'stronghold', 'researcher').
    """
    
    # --- Default Configuration ---
    DEFAULT_CODENAME_MAX_LEN = 3
    DEFAULT_TEAM_DIGITS = 4
    DEFAULT_STRONGHOLD_DIGITS = 3
    DEFAULT_RESEARCHER_DIGITS = 5

    def __init__(self, club: str, team: str | None = None, stronghold: str | None = None, researcher: str | None = None, **config):
        """
        Initializes a GSCUIdentifier instance. It's recommended to use factory
        methods like .from_string() or .from_int() for easier instantiation.
        """
        # --- Instance Configuration ---
        self.config = {
            'team_digits': config.get('team_digits', self.DEFAULT_TEAM_DIGITS),
            'stronghold_digits': config.get('stronghold_digits', self.DEFAULT_STRONGHOLD_DIGITS),
            'researcher_digits': config.get('researcher_digits', self.DEFAULT_RESEARCHER_DIGITS),
            'codename_max_len': config.get('codename_max_len', self.DEFAULT_CODENAME_MAX_LEN)
        }

        # --- Validate and Store Components ---
        if not club.isalpha() or len(club) < 2:
            raise ValueError("Club codename must be at least two alphabetic characters.")
        self._club = club.upper()

        self._team = self._validate_part(team, 'team')
        self._stronghold = self._validate_part(stronghold, 'stronghold')
        self._researcher = self._validate_part(researcher, 'researcher')
        
        # --- Determine and freeze object state ---
        self._level = self._determine_level()
        self._str_repr = self._build_string_representation()
        self._int_repr = self._build_integer_representation()
        
    def _validate_part(self, value: str | None, part_name: str) -> str | None:
        """Validates a numeric part of the identifier."""
        if value is None:
            return None
        
        expected_len = self.config[f'{part_name}_digits']
        if not value.isdigit() or len(value) != expected_len:
            raise ValueError(
                f"{part_name.capitalize()} ID must be a string of {expected_len} digits, but got '{value}'."
            )
        return value

    # --- Public Properties ---
    @property
    def club(self) -> str: return self._club
    @property
    def team(self) -> str | None: return self._team
    @property
    def stronghold(self) -> str | None: return self._stronghold
    @property
    def researcher(self) -> str | None: return self._researcher
    @property
    def level(self) -> str: return self._level
    @property
    def as_int(self) -> int: return self._int_repr

    # --- Factory Methods ---
    @classmethod
    def from_string(cls, code_str: str, **config) -> GSCUIdentifier:
        """Creates a GSCUIdentifier from a full string code."""
        parts = cls._parse_full_code(code_str, **config)
        return cls(*parts, **config)

    @classmethod
    def from_int(cls, num: int, level: str, **config) -> GSCUIdentifier:
        """Creates a GSCUIdentifier from an integer representation."""
        parts = cls._convert_int_to_full_code(num, level, **config)
        return cls(*parts, **config)
    
    @classmethod
    def from_bytes(cls, byte_seq: bytes, level: str, **config) -> GSCUIdentifier:
        """Creates a GSCUIdentifier from a bytes representation."""
        num = int.from_bytes(byte_seq, 'big')
        return cls.from_int(num, level, **config)

    # --- Public Methods ---
    def to_bytes(self, num_bytes: int | None = None) -> bytes:
        """
        Converts the identifier to a bytes object.
        If num_bytes is not provided, it will be calculated automatically.
        """
        _, min_bytes_req = self.get_min_storage_size()
        
        if num_bytes is None:
            num_bytes = min_bytes_req
        
        if num_bytes < min_bytes_req:
            raise ValueError(f"Identifier '{self}' requires at least {min_bytes_req} bytes for storage, but {num_bytes} were provided.")

        return self.as_int.to_bytes(num_bytes, 'big')

    def get_min_storage_size(self) -> Tuple[int, int]:
        """Calculates the minimum bits and bytes required to store this identifier."""
        return self._calculate_min_storage_size(
            self.level,
            **self.config
        )

    # --- Dunder Methods for rich object behavior ---
    def __str__(self) -> str:
        return self._str_repr

    def __repr__(self) -> str:
        config_str = ", ".join(f"{k}={v!r}" for k, v in self.config.items() if v != getattr(self, f"DEFAULT_{k.upper()}"))
        if config_str:
            return f"GSCUIdentifier.from_string('{self}', {config_str})"
        return f"GSCUIdentifier.from_string('{self}')"

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, GSCUIdentifier):
            return NotImplemented
        return self._str_repr == other._str_repr and self.config == other.config

    def __hash__(self) -> int:
        return hash((self._str_repr, tuple(self.config.items())))

    # --- Private Helper and Static Methods for Core Logic ---
    def _determine_level(self) -> str:
        """Determines the hierarchy level of the identifier."""
        if self._researcher: return 'researcher'
        if self._stronghold: return 'stronghold'
        if self._team: return 'team'
        return 'club'

    def _build_string_representation(self) -> str:
        """Constructs the canonical string representation."""
        if self.level == 'club':
            return self.club
        if self.level == 'team':
            return f"{self.club}{self.team}"
        if self.level == 'stronghold':
            return f"{self.club}{self.team}-{self.stronghold}"
        # Researcher level
        return f"{self.club}{self.team}-{self.stronghold}{self.researcher}"
        
    def _build_integer_representation(self) -> int:
        """Constructs the canonical integer representation."""
        club_int = self._convert_codename_to_int(self.club)
        if self.level == 'club':
            return club_int
        
        team_int = int(self.team)
        team_mul = 10**self.config['team_digits']
        current_val = club_int * team_mul + team_int
        if self.level == 'team':
            return current_val

        stronghold_int = int(self.stronghold)
        stronghold_mul = 10**self.config['stronghold_digits']
        current_val = current_val * stronghold_mul + stronghold_int
        if self.level == 'stronghold':
            return current_val
        
        researcher_int = int(self.researcher)
        researcher_mul = 10**self.config['researcher_digits']
        return current_val * researcher_mul + researcher_int

    @staticmethod
    def _calculate_min_storage_size(level: str, codename_max_len: int, team_digits: int, stronghold_digits: int, researcher_digits: int) -> Tuple[int, int]:
        """Calculates minimum storage size based on configuration."""
        log_2_10 = math.log(10, 2)
        
        if codename_max_len < 2:
             raise ValueError("Codename max length must be at least 2.")
        if codename_max_len < 10:
            codename_bits = math.log2((26**(codename_max_len + 1) - 676) // 25)
        else:
            codename_bits = (codename_max_len + 1) * math.log2(26) - math.log2(25)

        level_bits_map = {
            'club': codename_bits,
            'team': codename_bits + team_digits * log_2_10,
            'stronghold': codename_bits + (team_digits + stronghold_digits) * log_2_10,
            'researcher': codename_bits + (team_digits + stronghold_digits + researcher_digits) * log_2_10
        }
        
        bits = math.ceil(level_bits_map[level])
        bytes_val = math.ceil(bits / 8)
        return bits, bytes_val

    @staticmethod
    def _parse_full_code(code: str, **config) -> Tuple[str, ...]:
        """Parses a GSCU code string into its constituent parts."""
        cfg = {
            'team_digits': config.get('team_digits', GSCUIdentifier.DEFAULT_TEAM_DIGITS),
            'stronghold_digits': config.get('stronghold_digits', GSCUIdentifier.DEFAULT_STRONGHOLD_DIGITS),
            'researcher_digits': config.get('researcher_digits', GSCUIdentifier.DEFAULT_RESEARCHER_DIGITS)
        }
        
        i = 0
        for char in code:
            if char.isalpha(): i += 1
            else: break
        
        club, remainder = code[:i], code[i:]
        
        if not remainder: return (club,)
        
        team = remainder[:cfg['team_digits']]
        remainder = remainder[cfg['team_digits']:]
        if not remainder: return club, team
        
        if not remainder.startswith('-'):
            raise ValueError(f"Expected '-' separator after team ID in '{code}'.")
        
        remainder = remainder[1:]
        stronghold = remainder[:cfg['stronghold_digits']]
        remainder = remainder[cfg['stronghold_digits']:]
        if not remainder: return club, team, stronghold
        
        researcher = remainder[:cfg['researcher_digits']]
        return club, team, stronghold, researcher

    @staticmethod
    def _convert_codename_to_int(codename: str) -> int:
        """Converts an alphabetic codename to its unique integer ID."""
        value = 0
        cdlen = len(codename)
        if cdlen < 2:
            raise ValueError("Codename requires at least two letters.")
            
        offset = (26**cdlen - 676) // 25 if cdlen > 2 else 0
        
        for char in codename.upper():
            value = value * 26 + (ord(char) - 65)
            
        return value + offset

    @staticmethod
    def _convert_int_to_full_code(num: int, level: str, **config) -> Tuple[str, ...]:
        """Converts an integer back to its GSCU code parts based on level."""
        cfg = {
            'team_digits': config.get('team_digits', GSCUIdentifier.DEFAULT_TEAM_DIGITS),
            'stronghold_digits': config.get('stronghold_digits', GSCUIdentifier.DEFAULT_STRONGHOLD_DIGITS),
            'researcher_digits': config.get('researcher_digits', GSCUIdentifier.DEFAULT_RESEARCHER_DIGITS)
        }
        
        researcher, stronghold, team = None, None, None
        
        if level == 'researcher':
            researcher_div = 10**cfg['researcher_digits']
            researcher = f"{num % researcher_div:0{cfg['researcher_digits']}}"
            num //= researcher_div
        
        if level in ['researcher', 'stronghold']:
            stronghold_div = 10**cfg['stronghold_digits']
            stronghold = f"{num % stronghold_div:0{cfg['stronghold_digits']}}"
            num //= stronghold_div

        if level in ['researcher', 'stronghold', 'team']:
            team_div = 10**cfg['team_digits']
            team = f"{num % team_div:0{cfg['team_digits']}}"
            num //= team_div

        codename = GSCUIdentifier._convert_int_to_codename(num)
        
        parts = [p for p in (codename, team, stronghold, researcher) if p is not None]
        return tuple(parts)

    @staticmethod
    def _convert_int_to_codename(num: int) -> str:
        """Converts a club integer ID back to its alphabetic codename."""
        if num < 0:
            raise ValueError("Club integer ID cannot be negative.")
        if num < 676:
            codename_len = 2
            offset = 0
        else:
            val = num * 25 + 676
            codename_len = 0
            closest = 1
            while closest * 26 <= val:
                closest *= 26
                codename_len += 1
            offset = (26**codename_len - 676) // 25
        
        number = num - offset
        codename = ""
        for _ in range(codename_len):
            codename = chr(65 + number % 26) + codename
            number //= 26
        return codename

# --- Example Usage ---
if __name__ == "__main__":
    print("--- Standard GSCU Identifier Example ---")

    club_id = GSCUIdentifier.from_string("LFN")
    print(f"\nObject from string: {club_id!r}")
    print(f"  Club: {club_id.club}")
    print(f"  Level: {club_id.level}")
    print(f"  Integer value: {club_id.as_int}")
    
    club_bits, club_byte_len = club_id.get_min_storage_size()
    print(f"  Min storage: {club_bits} bits ({club_byte_len} bytes)")
    
    club_bytes = club_id.to_bytes()
    print(f"  Bytes value (auto-sized to {len(club_bytes)} bytes): {club_bytes.hex(' ')}")

    club_bytes_4 = club_id.to_bytes(4)
    print(f"  Bytes value (manually-sized to 4 bytes): {club_bytes_4.hex(' ')}")

    team_id = GSCUIdentifier.from_string("LFN2018")
    print(f"\nObject from string: {team_id!r}")
    print(f"  Club: {team_id.club}, Team: {team_id.team}")
    print(f"  Level: {team_id.level}")
    print(f"  Integer value: {team_id.as_int}")
    
    team_bits, team_byte_len = team_id.get_min_storage_size()
    print(f"  Min storage: {team_bits} bits ({team_byte_len} bytes)")
    
    team_bytes = team_id.to_bytes()
    print(f"  Bytes value (auto-sized to {len(team_bytes)} bytes): {team_bytes.hex(' ')}")

    team_bytes_4 = team_id.to_bytes(4)
    print(f"  Bytes value (manually-sized to 4 bytes): {team_bytes_4.hex(' ')}")

    stronghold_id = GSCUIdentifier.from_string("LFN2018-001")
    print(f"\nObject from string: {stronghold_id!r}")
    print(f"  Club: {stronghold_id.club}, Team: {stronghold_id.team}, Stronghold: {stronghold_id.stronghold}")
    print(f"  Level: {stronghold_id.level}")
    print(f"  Integer value: {stronghold_id.as_int}")
    
    stronghold_bits, stronghold_byte_len = stronghold_id.get_min_storage_size()
    print(f"  Min storage: {stronghold_bits} bits ({stronghold_byte_len} bytes)")
    
    stronghold_bytes = stronghold_id.to_bytes()
    print(f"  Bytes value (auto-sized to {len(stronghold_bytes)} bytes): {stronghold_bytes.hex(' ')}")

    stronghold_bytes_6 = stronghold_id.to_bytes(6)
    print(f"  Bytes value (manually-sized to 6 bytes): {stronghold_bytes_6.hex(' ')}")
    
    researcher_id = GSCUIdentifier.from_string("LFN2018-00121376")
    print(f"\nObject from string: {researcher_id!r}")
    print(f"  Club: {researcher_id.club}, Team: {researcher_id.team}, Stronghold: {researcher_id.stronghold}, Researcher: {researcher_id.researcher}")
    print(f"  Level: {researcher_id.level}")
    print(f"  Integer value: {researcher_id.as_int}")
    
    researcher_bits, researcher_byte_len = researcher_id.get_min_storage_size()
    print(f"  Min storage: {researcher_bits} bits ({researcher_byte_len} bytes)")
    
    researcher_bytes = researcher_id.to_bytes()
    print(f"  Bytes value (auto-sized to {len(researcher_bytes)} bytes): {researcher_bytes.hex(' ')}")

    researcher_bytes_8 = researcher_id.to_bytes(8)
    print(f"  Bytes value (manually-sized to 8 bytes): {researcher_bytes_8.hex(' ')}")
    
    recreated_id = GSCUIdentifier.from_bytes(researcher_bytes, level='researcher')
    print(f"Recreated from bytes: {recreated_id!r}")
    print(f"  Are they equal? {researcher_id == recreated_id}")

    print("\n--- Extended Configuration Example (4-letter codename, etc.) ---")
    
    ext_config = {
        'team_digits': 5,
        'stronghold_digits': 4,
        'researcher_digits': 6,
        'codename_max_len': 4
    }
    
    ext_id = GSCUIdentifier.from_string("ABCD12345-0001000001", **ext_config)
    print(f"Extended ID: {ext_id!r}")
    print(f"  Integer value: {ext_id.as_int}")
    
    bits, byte_len = ext_id.get_min_storage_size()
    print(f"  Min storage: {bits} bits ({byte_len} bytes)")
    
    ext_bytes = ext_id.to_bytes()
    print(f"  Bytes value: {ext_bytes.hex(' ')}")

    ext_bytes_10 = ext_id.to_bytes(10)
    print(f"  Bytes value (manually-sized to 10 bytes): {ext_bytes_10.hex(' ')}")

    ext_bytes_12 = ext_id.to_bytes(12)
    print(f"  Bytes value (manually-sized to 12 bytes): {ext_bytes_12.hex(' ')}")
    
    recreated_ext_id = GSCUIdentifier.from_bytes(ext_bytes, level='researcher', **ext_config)
    print(f"Recreated extended ID: {recreated_ext_id!r}")
    print(f"  Are they equal? {ext_id == recreated_ext_id}")

    print("\n--- Extremely Extended Configuration Example (26-letter codename/512-letter codename, etc.) ---")
    
    ext_26_config = {
        'team_digits': 20,
        'stronghold_digits': 20,
        'researcher_digits': 20,
        'codename_max_len': 26
    }
    
    ext_26_id = GSCUIdentifier.from_string("ABCDEFGHIJKLMNOPQRSTUVWXYZ12345678901234567890-1234567890123456789012345678901234567890", **ext_26_config)
    print(f"Extended ID: {ext_26_id!r}")
    print(f"  Integer value: {ext_26_id.as_int}")
    
    bits, byte_len = ext_26_id.get_min_storage_size()
    print(f"  Min storage: {bits} bits ({byte_len} bytes)")
    
    ext_26_bytes = ext_26_id.to_bytes()
    print(f"  Bytes value: {ext_26_bytes.hex(' ')}")
    
    recreated_ext_26_id = GSCUIdentifier.from_bytes(ext_26_bytes, level='researcher', **ext_26_config)
    print(f"Recreated extended ID: {recreated_ext_26_id!r}")
    print(f"  Are they equal? {ext_26_id == recreated_ext_26_id}")

    ext_512_config = {
        'team_digits': 768,
        'stronghold_digits': 384,
        'researcher_digits': 512,
        'codename_max_len': 512
    }
    
    ext_512_id = GSCUIdentifier.from_string("SCCYZPFOBRIQGZYGROFQRDKEETSGVENAIHXMEKXDVFLJKOGMXDZJNJFADLACXIZIDDKOLYZHPACEXGMSHNFVTKFUTHEHBLSOKIYIVIMJJEKOAKPDLNWWQXTLPSICKUFVZSHZPVAOESCKFQUETKVTVTRGYHDQBQXQVUSTFJEXFDTTHKFPMBQHJWNUJGUEZCHZLDVQAUJROMZCXOEQWQUGDMEMNXVHIODOBWXYBQRYMGKNTMBMEWNCSTCHFKPUAAJMOIYEPZUWPSIILEESQOZPKPFJFCETIPDPHRIGIMZPSFVDHWWUHXGHCDEKZJVDOJBQQQLHHXIKJYLCQCVYUWMSUQCJNFPLSFZUZHFPBOAGHAHCXDHXDHKTZCCIUQPIYQZLZJCSBDCWVSDKWSXPGHDFKISRFEIVBSNLPTHOWUFEIWIFLYULHPBYCKGQILXLTCNNCHRQWBLOEVZZPQSPXBJSQJKFRAXCWRJEFAINTWDNHTVPDEEYVLWLJITIPJGRIRRK816270624772780919451682435992341391608295131397399883007978652978802166296515852637022645720462112609470325168099091813094794447015366467479555944488923519184588488099642896816431667893486930459526964785526758583690311025802834027520981355416779466707116813405788843268914099624044200341578555998792568198953001337146948888574186095359461916375778911358230042062708686265345401723807229542385449250150069769271514179894680930820731534241957193985410646990247002416520434643532898310932499365952733444103887167167272094077549257959439954040898922812591867413227295997371994390969538128277454991367104223293092711913963089635614847982718867596810477847702070605605931927485011404683742856083841414317795920829483360488388320951494639393092230164674235824827518715228438-84969615826702367528886267075424080709640399051738180969188621916878381383129538870924446293079438699562242593259693045461480492851346921560410217197306848898166918183187631775870921758535536071259410226147021456446311176767619670988932840940576733000187848399407312076903769185048708692996004606817055360589218886336550418029951548475411383439087316611488421300574688522414066629344304762990175128784418931121927397909856651177848509070502745904774879363237256082759271052001375024322364263260741597129469852203802028581239924271891353312879487743048963670260661285416911984709395241451550844969423465989543787186709974824616588038257691687448528933691280710475346671257695685991667897376097193780021069309394488015404665784280366508548974946582160108107348880341636556232828887428104735214385226578026401228124722742762842510365430889392773097449468280295165758091093737581802543966178336191016", **ext_512_config)
    print(f"Extended ID: {ext_512_id!r}")
    print(f"  Integer value: {ext_512_id.as_int}")
    
    bits, byte_len = ext_512_id.get_min_storage_size()
    print(f"  Min storage: {bits} bits ({byte_len} bytes)")
    
    ext_512_bytes = ext_512_id.to_bytes()
    print(f"  Bytes value: {ext_512_bytes.hex(' ')}")
    
    recreated_ext_512_id = GSCUIdentifier.from_bytes(ext_512_bytes, level='researcher', **ext_512_config)
    print(f"Recreated extended ID: {recreated_ext_512_id!r}")
    print(f"  Are they equal? {ext_512_id == recreated_ext_512_id}")
```

运行结果：

```plaintext
--- Standard GSCU Identifier Example ---
Object from string: GSCUIdentifier.from_string('LFN')
  Club: LFN
  Level: club
  Integer value: 8255
  Min storage: 15 bits (2 bytes)
  Bytes value (auto-sized to 2 bytes): 20 3f
  Bytes value (manually-sized to 4 bytes): 00 00 20 3f
Object from string: GSCUIdentifier.from_string('LFN2018')
  Club: LFN, Team: 2018
  Level: team
  Integer value: 82552018
  Min storage: 28 bits (4 bytes)
  Bytes value (auto-sized to 4 bytes): 04 eb a4 d2
  Bytes value (manually-sized to 4 bytes): 04 eb a4 d2
Object from string: GSCUIdentifier.from_string('LFN2018-001')
  Club: LFN, Team: 2018, Stronghold: 001
  Level: stronghold
  Integer value: 82552018001
  Min storage: 38 bits (5 bytes)
  Bytes value (auto-sized to 5 bytes): 13 38 7b d4 51
  Bytes value (manually-sized to 6 bytes): 00 13 38 7b d4 51
Object from string: GSCUIdentifier.from_string('LFN2018-00121376')
  Club: LFN, Team: 2018, Stronghold: 001, Researcher: 21376
  Level: researcher
  Integer value: 8255201800121376
  Min storage: 55 bits (7 bytes)
  Bytes value (auto-sized to 7 bytes): 1d 54 0f f2 d8 6c 20
  Bytes value (manually-sized to 8 bytes): 00 1d 54 0f f2 d8 6c 20
Recreated from bytes: GSCUIdentifier.from_string('LFN2018-00121376')
  Are they equal? True
--- Extended Configuration Example (4-letter codename, etc.) ---
Extended ID: GSCUIdentifier.from_string('ABCD12345-0001000001', team_digits=5, stronghold_digits=4, researcher_digits=6, codename_max_len=4)
  Integer value: 18983123450001000001
  Min storage: 69 bits (9 bytes)
  Bytes value: 01 07 71 9a 33 6c b2 86 41
  Bytes value (manually-sized to 10 bytes): 00 01 07 71 9a 33 6c b2 86 41
  Bytes value (manually-sized to 12 bytes): 00 00 00 01 07 71 9a 33 6c b2 86 41
Recreated extended ID: GSCUIdentifier.from_string('ABCD12345-0001000001', team_digits=5, stronghold_digits=4, researcher_digits=6, codename_max_len=4)
  Are they equal? True
--- Extremely Extended Configuration Example (26-letter codename/512-letter codename, etc.) ---
Extended ID: GSCUIdentifier.from_string('ABCDEFGHIJKLMNOPQRSTUVWXYZ12345678901234567890-1234567890123456789012345678901234567890', team_digits=20, stronghold_digits=20, researcher_digits=20, codename_max_len=26)
  Integer value: 256094574536617744129141650397448449123456789012345678901234567890123456789012345678901234567890
  Min storage: 322 bits (41 bytes)
  Bytes value: 00 1e b1 73 86 15 c1 58 06 0f 08 13 ec 65 94 42 75 9a a3 ac 83 56 b0 f0 78 8f 17 c5 7c 26 f3 03 05 9c cf f1 96 ce 3f 0a d2
Recreated extended ID: GSCUIdentifier.from_string('ABCDEFGHIJKLMNOPQRSTUVWXYZ12345678901234567890-1234567890123456789012345678901234567890', team_digits=20, stronghold_digits=20, researcher_digits=20, codename_max_len=26)
  Are they equal? True
Extended ID: GSCUIdentifier.from_string('SCCYZPFOBRIQGZYGROFQRDKEETSGVENAIHXMEKXDVFLJKOGMXDZJNJFADLACXIZIDDKOLYZHPACEXGMSHNFVTKFUTHEHBLSOKIYIVIMJJEKOAKPDLNWWQXTLPSICKUFVZSHZPVAOESCKFQUETKVTVTRGYHDQBQXQVUSTFJEXFDTTHKFPMBQHJWNUJGUEZCHZLDVQAUJROMZCXOEQWQUGDMEMNXVHIODOBWXYBQRYMGKNTMBMEWNCSTCHFKPUAAJMOIYEPZUWPSIILEESQOZPKPFJFCETIPDPHRIGIMZPSFVDHWWUHXGHCDEKZJVDOJBQQQLHHXIKJYLCQCVYUWMSUQCJNFPLSFZUZHFPBOAGHAHCXDHXDHKTZCCIUQPIYQZLZJCSBDCWVSDKWSXPGHDFKISRFEIVBSNLPTHOWUFEIWIFLYULHPBYCKGQILXLTCNNCHRQWBLOEVZZPQSPXBJSQJKFRAXCWRJEFAINTWDNHTVPDEEYVLWLJITIPJGRIRRK816270624772780919451682435992341391608295131397399883007978652978802166296515852637022645720462112609470325168099091813094794447015366467479555944488923519184588488099642896816431667893486930459526964785526758583690311025802834027520981355416779466707116813405788843268914099624044200341578555998792568198953001337146948888574186095359461916375778911358230042062708686265345401723807229542385449250150069769271514179894680930820731534241957193985410646990247002416520434643532898310932499365952733444103887167167272094077549257959439954040898922812591867413227295997371994390969538128277454991367104223293092711913963089635614847982718867596810477847702070605605931927485011404683742856083841414317795920829483360488388320951494639393092230164674235824827518715228438-84969615826702367528886267075424080709640399051738180969188621916878381383129538870924446293079438699562242593259693045461480492851346921560410217197306848898166918183187631775870921758535536071259410226147021456446311176767619670988932840940576733000187848399407312076903769185048708692996004606817055360589218886336550418029951548475411383439087316611488421300574688522414066629344304762990175128784418931121927397909856651177848509070502745904774879363237256082759271052001375024322364263260741597129469852203802028581239924271891353312879487743048963670260661285416911984709395241451550844969423465989543787186709974824616588038257691687448528933691280710475346671257695685991667897376097193780021069309394488015404665784280366508548974946582160108107348880341636556232828887428104735214385226578026401228124722742762842510365430889392773097449468280295165758091093737581802543966178336191016', team_digits=768, stronghold_digits=384, researcher_digits=512, codename_max_len=512)
  Integer value: 2152277669617411707527228506614657545274393961758934268513055146236306579787175928933464170907097955045862799163382684421589504391164810942926125557900482075362912261217240809228824261730817583869078631679141566843961254883676525850762842502013998456664917471100217196992099561071666896346191437261386798091961754189179798735314070100576986236015035873416618930439572051430551333991040870137490498660500664240392657465560601799691246212434896293069138346541294699935924403613118824501150913623998010751232836907998566617621289416694493885642882171867709291028717247008241661145138514810643643878507552277712248132227301272186642432098517999908772152490813549212592841168148474225432519677385677454495563749122790493670259384481627062477278091945168243599234139160829513139739988300797865297880216629651585263702264572046211260947032516809909181309479444701536646747955594448892351918458848809964289681643166789348693045952696478552675858369031102580283402752098135541677946670711681340578884326891409962404420034157855599879256819895300133714694888857418609535946191637577891135823004206270868626534540172380722954238544925015006976927151417989468093082073153424195719398541064699024700241652043464353289831093249936595273344410388716716727209407754925795943995404089892281259186741322729599737199439096953812827745499136710422329309271191396308963561484798271886759681047784770207060560593192748501140468374285608384141431779592082948336048838832095149463939309223016467423582482751871522843884969615826702367528886267075424080709640399051738180969188621916878381383129538870924446293079438699562242593259693045461480492851346921560410217197306848898166918183187631775870921758535536071259410226147021456446311176767619670988932840940576733000187848399407312076903769185048708692996004606817055360589218886336550418029951548475411383439087316611488421300574688522414066629344304762990175128784418931121927397909856651177848509070502745904774879363237256082759271052001375024322364263260741597129469852203802028581239924271891353312879487743048963670260661285416911984709395241451550844969423465989543787186709974824616588038257691687448528933691280710475346671257695685991667897376097193780021069309394488015404665784280366508548974946582160108107348880341636556232828887428104735214385226578026401228124722742762842510365430889392773097449468280295165758091093737581802543966178336191016
  Min storage: 7935 bits (992 bytes)
  Bytes value: 3a 7d d2 35 d8 9d a8 e1 49 82 1a 00 6b 24 62 95 6b c0 f1 c2 15 4e 3d 65 a5 e7 40 60 0e 93 48 a1 d4 c5 a7 f8 8b 0c 29 83 67 3c 91 6a d0 2d 79 cd 9b ee 45 99 42 4f 0e b5 45 4c 24 d0 64 0a 89 13 48 3c 73 11 a9 70 ea f4 6b 94 22 e6 0a 99 19 54 b4 24 e1 0d 2b bf b7 f5 1a 3a 4a 6d e3 ae 3d 34 e2 3c e2 3e 82 36 fd d2 ec 43 af b3 86 74 c3 cd b7 a2 78 ec 55 64 27 29 8e 34 2c b4 8a ee af 76 65 09 00 0d 19 e7 9c f3 dc a6 bd 2d 6f 91 74 a4 be ff 95 58 3f 26 04 94 14 d4 65 8a c2 32 f1 e9 50 2d cd a9 ec ff ff c1 b9 ba ea 2f ce 88 e8 16 bc 5a 72 cd 8e e5 57 5a cd 79 07 2f 4f 4c de 48 a7 90 52 c3 9b aa d1 71 76 80 f5 96 a6 91 22 26 1d 22 71 4a 9b d8 16 9d 5d 05 f4 8c 53 b9 6f 5e 6b e3 67 b9 b7 96 0a 10 cb e8 75 e9 36 f0 41 28 1d ad 15 54 f5 c1 6b 23 12 1f e1 32 c7 45 d2 d2 17 a2 f4 97 84 b4 a2 60 32 07 97 cd 65 c1 f5 2c c4 8d ed 56 11 4d 86 8c 12 eb ec e7 76 4f 51 f1 30 bc 68 a1 51 46 ed 88 10 72 2d 03 48 21 a1 f6 c8 6f 6a c7 39 ee b0 4d 67 49 a0 09 7c 62 16 24 65 23 ff 5a 96 0f 19 33 a3 41 71 61 48 30 41 24 ca 4d c7 bf 3b 55 91 73 65 7a be 59 be ce c5 2b 5a 4c 87 31 03 f5 93 15 72 4f 41 1b e8 bc 86 2e b2 35 5b a4 25 2d 9c a4 81 00 da 70 30 3c 9d 3e 24 56 c0 d1 71 29 ad 38 33 3d 2d 0b 52 20 6e 12 c7 7e cd 10 fe be 2e c5 0f b7 3c 27 4d b1 96 54 19 57 09 a2 7c bf e0 64 02 c6 be 6f 96 60 d7 8d e4 be 20 3c 7f 98 a8 22 cb 4a e0 67 e0 34 e4 4d 64 6b 7c 40 55 d1 29 2d f5 ac 94 9d 17 09 8a d2 b2 5d e2 3f 4a 7c 44 9b 8b c4 7c bc 9d 4c d7 f2 14 0c ff 0f 8a 81 7e 24 24 5e bb 8c a8 4e b3 ef 6d 73 94 0f a0 bd 86 bf 39 dd 9f 9a e0 48 55 11 5a 9c 42 00 fa 70 00 9d e6 f8 7b dc 18 9e 3d 4a f7 c8 a1 11 3e 36 bf 98 f3 5c db fe f3 d2 c6 c3 72 bc c9 70 78 04 ea de a0 77 6f a3 66 6d 26 de 94 e9 25 fc 7e 51 b0 c6 21 46 f9 0f e8 34 69 51 1c 23 20 e4 c5 a8 23 8b 50 f3 95 98 3a 4d 12 c8 ab f7 b8 87 55 15 c7 ed 2e c7 31 67 ca e3 08 75 ac 1b f8 53 99 cd 0b e5 bb e7 b1 b9 9d 81 b8 f7 83 28 c4 88 c9 a2 8b d1 83 9b c3 49 33 eb 64 60 68 2b 2e ea e7 c8 f1 3f 84 3e 1f 69 4e a6 3e f6 e3 df 8d ff ee 86 75 52 7c ae ed 3e c7 13 40 f3 a4 66 56 0d 8c ec 4a b4 c2 10 4e d6 41 3d a2 8c b5 53 4d b9 b7 28 23 7c e9 ce 01 c6 15 1a e8 3a af aa 81 04 5d 99 2d 47 34 b3 80 3c 4f 1d b4 53 4a 37 12 1d 2e 79 96 ba 40 36 88 dd 97 dc a8 1a 6a 1e 18 96 62 7d e1 aa cf 22 c6 9b cc 29 03 ff 87 6a 41 ad b3 28 96 62 ef 4e a5 a8 32 b8 08 b2 9e c3 87 4c 4f bf 1c ff dc c1 a2 4c 1a 96 99 c4 19 2a f2 27 39 3d 88 b6 ff 78 6f 9f ae cb 2a 16 d6 50 90 6b 90 3f 3b ca d3 b5 bd e9 c1 3e cb d3 9d d3 7b 9d 18 7f b1 b6 46 76 e0 04 e0 a0 9e d3 bd d5 cb 65 76 ec fd db 5e c3 e5 7c c5 03 55 6d 76 92 40 ce 95 aa b7 f5 48 ec ac 78 70 d0 23 bc c4 f7 a9 c4 ae 5a 27 86 00 2c a9 71 c9 bd 55 75 40 57 8d d5 86 6c ce 2e d4 66 60 3d e5 4a 49 08 a7 58 b6 10 74 a2 d4 85 05 b9 21 3f 7e 98 ee 2b 70 25 a3 84 29 6f 99 35 b5 7b e4 0e 9f a8 98 e6 e7 8b 42 55 5c e9 96 9f 83 a8 7b 3a 83 58 98 dd f4 77 88 2f 43 ab 6e 72 c2 d4 7f fd 3e e9 13 be c6 97 b3 cb e5 16 66 fc 2d c6 04 a0 f3 ad 38 83 8c a6 b8 bf 22 ae 6a 83 d1 ba 12 2d 9e f6 32 28
Recreated extended ID: GSCUIdentifier.from_string('SCCYZPFOBRIQGZYGROFQRDKEETSGVENAIHXMEKXDVFLJKOGMXDZJNJFADLACXIZIDDKOLYZHPACEXGMSHNFVTKFUTHEHBLSOKIYIVIMJJEKOAKPDLNWWQXTLPSICKUFVZSHZPVAOESCKFQUETKVTVTRGYHDQBQXQVUSTFJEXFDTTHKFPMBQHJWNUJGUEZCHZLDVQAUJROMZCXOEQWQUGDMEMNXVHIODOBWXYBQRYMGKNTMBMEWNCSTCHFKPUAAJMOIYEPZUWPSIILEESQOZPKPFJFCETIPDPHRIGIMZPSFVDHWWUHXGHCDEKZJVDOJBQQQLHHXIKJYLCQCVYUWMSUQCJNFPLSFZUZHFPBOAGHAHCXDHXDHKTZCCIUQPIYQZLZJCSBDCWVSDKWSXPGHDFKISRFEIVBSNLPTHOWUFEIWIFLYULHPBYCKGQILXLTCNNCHRQWBLOEVZZPQSPXBJSQJKFRAXCWRJEFAINTWDNHTVPDEEYVLWLJITIPJGRIRRK816270624772780919451682435992341391608295131397399883007978652978802166296515852637022645720462112609470325168099091813094794447015366467479555944488923519184588488099642896816431667893486930459526964785526758583690311025802834027520981355416779466707116813405788843268914099624044200341578555998792568198953001337146948888574186095359461916375778911358230042062708686265345401723807229542385449250150069769271514179894680930820731534241957193985410646990247002416520434643532898310932499365952733444103887167167272094077549257959439954040898922812591867413227295997371994390969538128277454991367104223293092711913963089635614847982718867596810477847702070605605931927485011404683742856083841414317795920829483360488388320951494639393092230164674235824827518715228438-84969615826702367528886267075424080709640399051738180969188621916878381383129538870924446293079438699562242593259693045461480492851346921560410217197306848898166918183187631775870921758535536071259410226147021456446311176767619670988932840940576733000187848399407312076903769185048708692996004606817055360589218886336550418029951548475411383439087316611488421300574688522414066629344304762990175128784418931121927397909856651177848509070502745904774879363237256082759271052001375024322364263260741597129469852203802028581239924271891353312879487743048963670260661285416911984709395241451550844969423465989543787186709974824616588038257691687448528933691280710475346671257695685991667897376097193780021069309394488015404665784280366508548974946582160108107348880341636556232828887428104735214385226578026401228124722742762842510365430889392773097449468280295165758091093737581802543966178336191016', team_digits=768, stronghold_digits=384, researcher_digits=512, codename_max_len=512)
  Are they equal? True
```

C++：

```cpp
// 编译时需要链接 GMP 库: g++ gscu_id.cpp -std=c++17 -o gscu_id -lgmp -lgmpxx
#include <iostream>

#include <iomanip>

#include <string>

#include <string_view>

#include <vector>

#include <stdexcept>

#include <algorithm>

#include <cctype>

#include <cstddef>

#include <gmpxx.h>

#include <sstream>

// --- 1. 配置结构体 ---
struct GSCUConfig {
    static constexpr int DEFAULT_TEAM_DIGITS = 4;
    static constexpr int DEFAULT_STRONGHOLD_DIGITS = 3;
    static constexpr int DEFAULT_RESEARCHER_DIGITS = 5;

    int teamDigits = DEFAULT_TEAM_DIGITS;
    int strongholdDigits = DEFAULT_STRONGHOLD_DIGITS;
    int researcherDigits = DEFAULT_RESEARCHER_DIGITS;
};

// --- 2. 主类 ---
class GSCUIdentifier {
    public:
        // --- 静态工厂方法 ---
        static GSCUIdentifier fromString(const std::string & code,
            const GSCUConfig & config = {}) {
            std::string_view code_sv(code);

            size_t club_end = 0;
            while (club_end < code_sv.length() && std::isalpha(static_cast < unsigned char > (code_sv[club_end]))) {
                club_end++;
            }
            if (club_end < 2) throw std::invalid_argument("Club codename must be at least two letters.");
            std::string club(code_sv.substr(0, club_end));
            code_sv.remove_prefix(club_end);

            std::string team, stronghold, researcher;

            if (!code_sv.empty()) {
                if (code_sv.length() < config.teamDigits) throw std::invalid_argument("Team ID is too short.");
                team = code_sv.substr(0, config.teamDigits);
                if (!std::all_of(team.begin(), team.end(), ::isdigit)) throw std::invalid_argument("Team ID contains non-digit characters.");
                code_sv.remove_prefix(config.teamDigits);
            }

            if (!code_sv.empty()) {
                if (code_sv.front() != '-') throw std::invalid_argument("Expected '-' separator after team ID.");
                code_sv.remove_prefix(1);
                if (code_sv.length() < config.strongholdDigits) throw std::invalid_argument("Stronghold ID is too short.");
                stronghold = code_sv.substr(0, config.strongholdDigits);
                if (!std::all_of(stronghold.begin(), stronghold.end(), ::isdigit)) throw std::invalid_argument("Stronghold ID contains non-digit characters.");
                code_sv.remove_prefix(config.strongholdDigits);
            }

            if (!code_sv.empty()) {
                if (code_sv.length() != config.researcherDigits)
                    throw std::invalid_argument("Researcher ID has incorrect length or trailing characters.");
                researcher = code_sv.substr(0, config.researcherDigits);
                if (!std::all_of(researcher.begin(), researcher.end(), ::isdigit))
                    throw std::invalid_argument("Researcher ID contains non-digit characters.");
            }

            return GSCUIdentifier(club, team, stronghold, researcher, config);
        }

    static GSCUIdentifier fromInteger(const mpz_class & num,
        const std::string & level,
            const GSCUConfig & config = {}) {
        mpz_class temp_num = num;
        std::string researcher, stronghold, team;

        auto safe_pad = [](std::string s, int len) {
            if (s.length() < len) {
                s.insert(0, len - s.length(), '0');
            }
            return s;
        };

        if (level == "researcher") {
            mpz_class div = power(10, config.researcherDigits);
            mpz_class rem = temp_num % div;
            researcher = safe_pad(rem.get_str(), config.researcherDigits);
            temp_num /= div;
        }
        if (level == "researcher" || level == "stronghold") {
            mpz_class div = power(10, config.strongholdDigits);
            mpz_class rem = temp_num % div;
            stronghold = safe_pad(rem.get_str(), config.strongholdDigits);
            temp_num /= div;
        }
        if (level == "researcher" || level == "stronghold" || level == "team") {
            mpz_class div = power(10, config.teamDigits);
            mpz_class rem = temp_num % div;
            team = safe_pad(rem.get_str(), config.teamDigits);
            temp_num /= div;
        }
        std::string club = intToCodename(temp_num);
        return GSCUIdentifier(club, team, stronghold, researcher, config);
    }

    mpz_class toInteger() const {
        mpz_class clubInt = codenameToInt(m_club);
        if (m_level == "club") return clubInt;

        // 显式指定 base = 10
        mpz_class teamVal(m_team, 10);
        mpz_class teamInt = clubInt * power(10, m_config.teamDigits) + teamVal;
        if (m_level == "team") return teamInt;

        // 显式指定 base = 10
        mpz_class strongholdVal(m_stronghold, 10);
        mpz_class strongholdInt = teamInt * power(10, m_config.strongholdDigits) + strongholdVal;
        if (m_level == "stronghold") return strongholdInt;

        // 显式指定 base = 10
        mpz_class researcherVal(m_researcher, 10);
        mpz_class researcherInt = strongholdInt * power(10, m_config.researcherDigits) + researcherVal;
        return researcherInt;
    }

    std::string toString() const {
        if (m_level == "club") return m_club;
        if (m_level == "team") return m_club + m_team;
        if (m_level == "stronghold") return m_club + m_team + "-" + m_stronghold;
        return m_club + m_team + "-" + m_stronghold + m_researcher;
    }

    std::vector < std::byte > toBytes_extend(int num_bytes) const {
        mpz_class val = this -> toInteger();
        size_t min_bytes = mpz_sizeinbase(val.get_mpz_t(), 256);
        if (val == 0 && min_bytes == 0) min_bytes = 1;

        if (num_bytes < min_bytes) {
            throw std::invalid_argument("Provided byte length is too small.");
        }

        std::vector < std::byte > bytes(num_bytes, std::byte {
            0
        });
        mpz_export(bytes.data() + (num_bytes - min_bytes), nullptr, 1, 1, 1, 0, val.get_mpz_t());
        return bytes;
    }

    const std::string & getLevel() const {
        return m_level;
    }

    private: GSCUIdentifier(std::string club, std::string team, std::string stronghold, std::string researcher, GSCUConfig config): m_club(std::move(club)),
    m_team(std::move(team)),
    m_stronghold(std::move(stronghold)),
    m_researcher(std::move(researcher)),
    m_config(std::move(config)) {
        std::transform(m_club.begin(), m_club.end(), m_club.begin(), ::toupper);
        if (!m_researcher.empty()) m_level = "researcher";
        else if (!m_stronghold.empty()) m_level = "stronghold";
        else if (!m_team.empty()) m_level = "team";
        else m_level = "club";
    }

    static mpz_class codenameToInt(const std::string & codename) {
        if (codename.length() < 2) throw std::invalid_argument("Codename must have at least 2 letters.");
        mpz_class value = 0;
        for (char ch: codename) value = value * 26 + (std::toupper(static_cast < unsigned char > (ch)) - 'A');
        if (codename.length() > 2) {
            mpz_class offset = power(26, codename.length()) - 676;
            offset /= 25;
            value += offset;
        }
        return value;
    }

    static std::string intToCodename(const mpz_class & n) {
        if (n < 0) throw std::invalid_argument("Integer cannot be negative.");
        size_t len;
        if (n < 676) {
            len = 2;
        } else {
            len = 2;
            while (true) {
                mpz_class next_offset_start = (power(26, len + 1) - 676) / 25;
                if (n < next_offset_start) break;
                len++;
            }
        }
        mpz_class offset = (len > 2) ? (power(26, len) - 676) / 25 : mpz_class(0);
        mpz_class number = n - offset;
        std::string codename = "";
        for (size_t i = 0; i < len; ++i) {
            mpz_class rem = number % 26;
            codename += static_cast < char > ('A' + rem.get_si());
            number /= 26;
        }
        std::reverse(codename.begin(), codename.end());
        return codename;
    }

    static mpz_class power(int base, int exp) {
        mpz_class result;
        mpz_ui_pow_ui(result.get_mpz_t(), base, exp);
        return result;
    }

    std::string m_club;
    std::string m_team;
    std::string m_stronghold;
    std::string m_researcher;
    std::string m_level;
    GSCUConfig m_config;
};

void printBytes(const std::vector < std::byte > & bytes) {
    std::stringstream ss;
    for (const auto & byte: bytes) {
        ss << std::setw(2) << std::setfill('0') << std::hex << static_cast < int > (byte) << " ";
    }
    std::cout << ss.str() << std::dec << std::endl;
}

void run_test(const std::string & code_str, int extend_bytes,
    const GSCUConfig & config = {}) {
    std::cout << "--- Testing: " << code_str << " ---" << std::endl;
    try {
        auto id = GSCUIdentifier::fromString(code_str, config);
        std::cout << "Parsed OK. Level: " << id.getLevel() << std::endl;
        mpz_class int_val = id.toInteger();
        std::cout << "Integer value: " << int_val.get_str() << std::endl;

        auto bytes = id.toBytes_extend(extend_bytes);
        std::cout << "Bytes (" << extend_bytes << " bytes): ";
        printBytes(bytes);

        auto recreated_id = GSCUIdentifier::fromInteger(int_val, id.getLevel(), config);
        std::cout << "Recreated from Integer: " << recreated_id.toString() << std::endl;

        if (id.toString() == recreated_id.toString()) {
            std::cout << "Verification successful!" << std::endl;
        } else {
            std::cout << "Verification FAILED!" << std::endl;
        }
    } catch (const std::exception & e) {
        std::cerr << "Error: " << e.what() << std::endl;
    }
    std::cout << std::endl;
}

int main() {
    run_test("LFN", 2);
    run_test("LFN2018", 4);
    run_test("LFN2018-001", 6);
    run_test("LFN2018-00121376", 8);

    GSCUConfig ext_config;
    ext_config.teamDigits = 5;
    ext_config.strongholdDigits = 4;
    ext_config.researcherDigits = 6;

    run_test("ABCD12345-1234123456", 10, ext_config);
    run_test("ABCD12345-0079009999", 10, ext_config);
    run_test("ABCD12345-0001000001", 10, ext_config);
    run_test("ABCD12345-1234123456", 12, ext_config);
    run_test("ABCD12345-0079009999", 12, ext_config);
    run_test("ABCD12345-0073009993", 12, ext_config);
    run_test("ABCD12345-0073009981", 12, ext_config);
    run_test("ABCD12345-0073009902", 12, ext_config);
    run_test("ABCD12345-0073009652", 12, ext_config);
    run_test("ABCD12345-0073009023", 12, ext_config);
    run_test("ABCD12345-0073008711", 12, ext_config);
    run_test("ABCD12345-0073004250", 12, ext_config);
    run_test("ABCD12345-0073001866", 12, ext_config);
    run_test("ABCD12345-0073000935", 12, ext_config);
    run_test("ABCD12345-0025002655", 12, ext_config);
    run_test("ABCD12345-0001000001", 12, ext_config);
 
 // 极端测试用例：512字母代号、768位队编号、384位据点编号、512位研究员编号
 GSCUConfig extreme_config;
    extreme_config.teamDigits = 768;
    extreme_config.strongholdDigits = 384;
    extreme_config.researcherDigits = 512;
 run_test("SCCYZPFOBRIQGZYGROFQRDKEETSGVENAIHXMEKXDVFLJKOGMXDZJNJFADLACXIZIDDKOLYZHPACEXGMSHNFVTKFUTHEHBLSOKIYIVIMJJEKOAKPDLNWWQXTLPSICKUFVZSHZPVAOESCKFQUETKVTVTRGYHDQBQXQVUSTFJEXFDTTHKFPMBQHJWNUJGUEZCHZLDVQAUJROMZCXOEQWQUGDMEMNXVHIODOBWXYBQRYMGKNTMBMEWNCSTCHFKPUAAJMOIYEPZUWPSIILEESQOZPKPFJFCETIPDPHRIGIMZPSFVDHWWUHXGHCDEKZJVDOJBQQQLHHXIKJYLCQCVYUWMSUQCJNFPLSFZUZHFPBOAGHAHCXDHXDHKTZCCIUQPIYQZLZJCSBDCWVSDKWSXPGHDFKISRFEIVBSNLPTHOWUFEIWIFLYULHPBYCKGQILXLTCNNCHRQWBLOEVZZPQSPXBJSQJKFRAXCWRJEFAINTWDNHTVPDEEYVLWLJITIPJGRIRRK816270624772780919451682435992341391608295131397399883007978652978802166296515852637022645720462112609470325168099091813094794447015366467479555944488923519184588488099642896816431667893486930459526964785526758583690311025802834027520981355416779466707116813405788843268914099624044200341578555998792568198953001337146948888574186095359461916375778911358230042062708686265345401723807229542385449250150069769271514179894680930820731534241957193985410646990247002416520434643532898310932499365952733444103887167167272094077549257959439954040898922812591867413227295997371994390969538128277454991367104223293092711913963089635614847982718867596810477847702070605605931927485011404683742856083841414317795920829483360488388320951494639393092230164674235824827518715228438-84969615826702367528886267075424080709640399051738180969188621916878381383129538870924446293079438699562242593259693045461480492851346921560410217197306848898166918183187631775870921758535536071259410226147021456446311176767619670988932840940576733000187848399407312076903769185048708692996004606817055360589218886336550418029951548475411383439087316611488421300574688522414066629344304762990175128784418931121927397909856651177848509070502745904774879363237256082759271052001375024322364263260741597129469852203802028581239924271891353312879487743048963670260661285416911984709395241451550844969423465989543787186709974824616588038257691687448528933691280710475346671257695685991667897376097193780021069309394488015404665784280366508548974946582160108107348880341636556232828887428104735214385226578026401228124722742762842510365430889392773097449468280295165758091093737581802543966178336191016", 1024, extreme_config);

    return 0;
}
```

运行结果：

```plaintext
--- Testing: LFN ---
Parsed OK. Level: club
Integer value: 8255
Bytes (2 bytes): 20 3f
Recreated from Integer: LFN
Verification successful!
--- Testing: LFN2018 ---
Parsed OK. Level: team
Integer value: 82552018
Bytes (4 bytes): 04 eb a4 d2
Recreated from Integer: LFN2018
Verification successful!
--- Testing: LFN2018-001 ---
Parsed OK. Level: stronghold
Integer value: 82552018001
Bytes (6 bytes): 00 13 38 7b d4 51
Recreated from Integer: LFN2018-001
Verification successful!
--- Testing: LFN2018-00121376 ---
Parsed OK. Level: researcher
Integer value: 8255201800121376
Bytes (8 bytes): 00 1d 54 0f f2 d8 6c 20
Recreated from Integer: LFN2018-00121376
Verification successful!
--- Testing: ABCD12345-1234123456 ---
Parsed OK. Level: researcher
Integer value: 18983123451234123456
Bytes (10 bytes): 00 01 07 71 9a 33 b6 32 7e c0
Recreated from Integer: ABCD12345-1234123456
Verification successful!
--- Testing: ABCD12345-0079009999 ---
Parsed OK. Level: researcher
Integer value: 18983123450079009999
Bytes (10 bytes): 00 01 07 71 9a 33 71 58 dc cf
Recreated from Integer: ABCD12345-0079009999
Verification successful!
--- Testing: ABCD12345-0001000001 ---
Parsed OK. Level: researcher
Integer value: 18983123450001000001
Bytes (10 bytes): 00 01 07 71 9a 33 6c b2 86 41
Recreated from Integer: ABCD12345-0001000001
Verification successful!
--- Testing: ABCD12345-1234123456 ---
Parsed OK. Level: researcher
Integer value: 18983123451234123456
Bytes (12 bytes): 00 00 00 01 07 71 9a 33 b6 32 7e c0
Recreated from Integer: ABCD12345-1234123456
Verification successful!
--- Testing: ABCD12345-0079009999 ---
Parsed OK. Level: researcher
Integer value: 18983123450079009999
Bytes (12 bytes): 00 00 00 01 07 71 9a 33 71 58 dc cf
Recreated from Integer: ABCD12345-0079009999
Verification successful!
--- Testing: ABCD12345-0073009993 ---
Parsed OK. Level: researcher
Integer value: 18983123450073009993
Bytes (12 bytes): 00 00 00 01 07 71 9a 33 70 fd 4f 49
Recreated from Integer: ABCD12345-0073009993
Verification successful!
--- Testing: ABCD12345-0073009981 ---
Parsed OK. Level: researcher
Integer value: 18983123450073009981
Bytes (12 bytes): 00 00 00 01 07 71 9a 33 70 fd 4f 3d
Recreated from Integer: ABCD12345-0073009981
Verification successful!
--- Testing: ABCD12345-0073009902 ---
Parsed OK. Level: researcher
Integer value: 18983123450073009902
Bytes (12 bytes): 00 00 00 01 07 71 9a 33 70 fd 4e ee
Recreated from Integer: ABCD12345-0073009902
Verification successful!
--- Testing: ABCD12345-0073009652 ---
Parsed OK. Level: researcher
Integer value: 18983123450073009652
Bytes (12 bytes): 00 00 00 01 07 71 9a 33 70 fd 4d f4
Recreated from Integer: ABCD12345-0073009652
Verification successful!
--- Testing: ABCD12345-0073009023 ---
Parsed OK. Level: researcher
Integer value: 18983123450073009023
Bytes (12 bytes): 00 00 00 01 07 71 9a 33 70 fd 4b 7f
Recreated from Integer: ABCD12345-0073009023
Verification successful!
--- Testing: ABCD12345-0073008711 ---
Parsed OK. Level: researcher
Integer value: 18983123450073008711
Bytes (12 bytes): 00 00 00 01 07 71 9a 33 70 fd 4a 47
Recreated from Integer: ABCD12345-0073008711
Verification successful!
--- Testing: ABCD12345-0073004250 ---
Parsed OK. Level: researcher
Integer value: 18983123450073004250
Bytes (12 bytes): 00 00 00 01 07 71 9a 33 70 fd 38 da
Recreated from Integer: ABCD12345-0073004250
Verification successful!
--- Testing: ABCD12345-0073001866 ---
Parsed OK. Level: researcher
Integer value: 18983123450073001866
Bytes (12 bytes): 00 00 00 01 07 71 9a 33 70 fd 2f 8a
Recreated from Integer: ABCD12345-0073001866
Verification successful!
--- Testing: ABCD12345-0073000935 ---
Parsed OK. Level: researcher
Integer value: 18983123450073000935
Bytes (12 bytes): 00 00 00 01 07 71 9a 33 70 fd 2b e7
Recreated from Integer: ABCD12345-0073000935
Verification successful!
--- Testing: ABCD12345-0025002655 ---
Parsed OK. Level: researcher
Integer value: 18983123450025002655
Bytes (12 bytes): 00 00 00 01 07 71 9a 33 6e 20 c6 9f
Recreated from Integer: ABCD12345-0025002655
Verification successful!
--- Testing: ABCD12345-0001000001 ---
Parsed OK. Level: researcher
Integer value: 18983123450001000001
Bytes (12 bytes): 00 00 00 01 07 71 9a 33 6c b2 86 41
Recreated from Integer: ABCD12345-0001000001
Verification successful!
--- Testing: SCCYZPFOBRIQGZYGROFQRDKEETSGVENAIHXMEKXDVFLJKOGMXDZJNJFADLACXIZIDDKOLYZHPACEXGMSHNFVTKF
UTHEHBLSOKIYIVIMJJEKOAKPDLNWWQXTLPSICKUFVZSHZPVAOESCKFQUETKVTVTRGYHDQBQXQVUSTFJEXFDTTHKFPMBQHJWNUJGU
EZCHZLDVQAUJROMZCXOEQWQUGDMEMNXVHIODOBWXYBQRYMGKNTMBMEWNCSTCHFKPUAAJMOIYEPZUWPSIILEESQOZPKPFJFCETIPD
PHRIGIMZPSFVDHWWUHXGHCDEKZJVDOJBQQQLHHXIKJYLCQCVYUWMSUQCJNFPLSFZUZHFPBOAGHAHCXDHXDHKTZCCIUQPIYQZLZJC
SBDCWVSDKWSXPGHDFKISRFEIVBSNLPTHOWUFEIWIFLYULHPBYCKGQILXLTCNNCHRQWBLOEVZZPQSPXBJSQJKFRAXCWRJEFAINTWD
NHTVPDEEYVLWLJITIPJGRIRRK816270624772780919451682435992341391608295131397399883007978652978802166296
5158526370226457204621126094703251680990918130947944470153664674795559444889235191845884880996428968
1643166789348693045952696478552675858369031102580283402752098135541677946670711681340578884326891409
9624044200341578555998792568198953001337146948888574186095359461916375778911358230042062708686265345
4017238072295423854492501500697692715141798946809308207315342419571939854106469902470024165204346435
3289831093249936595273344410388716716727209407754925795943995404089892281259186741322729599737199439
0969538128277454991367104223293092711913963089635614847982718867596810477847702070605605931927485011
404683742856083841414317795920829483360488388320951494639393092230164674235824827518715228438-849696
1582670236752888626707542408070964039905173818096918862191687838138312953887092444629307943869956224
2593259693045461480492851346921560410217197306848898166918183187631775870921758535536071259410226147
0214564463111767676196709889328409405767330001878483994073120769037691850487086929960046068170553605
8921888633655041802995154847541138343908731661148842130057468852241406662934430476299017512878441893
1121927397909856651177848509070502745904774879363237256082759271052001375024322364263260741597129469
8522038020285812399242718913533128794877430489636702606612854169119847093952414515508449694234659895
4378718670997482461658803825769168744852893369128071047534667125769568599166789737609719378002106930
9394488015404665784280366508548974946582160108107348880341636556232828887428104735214385226578026401
228124722742762842510365430889392773097449468280295165758091093737581802543966178336191016 ---
Parsed OK. Level: researcher
Integer value: 2152277669617411707527228506614657545274393961758934268513055146236306579787175928933
4641709070979550458627991633826844215895043911648109429261255579004820753629122612172408092288242617
3081758386907863167914156684396125488367652585076284250201399845666491747110021719699209956107166689
6346191437261386798091961754189179798735314070100576986236015035873416618930439572051430551333991040
8701374904986605006642403926574655606017996912462124348962930691383465412946999359244036131188245011
5091362399801075123283690799856661762128941669449388564288217186770929102871724700824166114513851481
0643643878507552277712248132227301272186642432098517999908772152490813549212592841168148474225432519
6773856774544955637491227904936702593844816270624772780919451682435992341391608295131397399883007978
6529788021662965158526370226457204621126094703251680990918130947944470153664674795559444889235191845
8848809964289681643166789348693045952696478552675858369031102580283402752098135541677946670711681340
5788843268914099624044200341578555998792568198953001337146948888574186095359461916375778911358230042
0627086862653454017238072295423854492501500697692715141798946809308207315342419571939854106469902470
0241652043464353289831093249936595273344410388716716727209407754925795943995404089892281259186741322
7295997371994390969538128277454991367104223293092711913963089635614847982718867596810477847702070605
6059319274850114046837428560838414143177959208294833604883883209514946393930922301646742358248275187
1522843884969615826702367528886267075424080709640399051738180969188621916878381383129538870924446293
0794386995622425932596930454614804928513469215604102171973068488981669181831876317758709217585355360
7125941022614702145644631117676761967098893284094057673300018784839940731207690376918504870869299600
4606817055360589218886336550418029951548475411383439087316611488421300574688522414066629344304762990
1751287844189311219273979098566511778485090705027459047748793632372560827592710520013750243223642632
6074159712946985220380202858123992427189135331287948774304896367026066128541691198470939524145155084
4969423465989543787186709974824616588038257691687448528933691280710475346671257695685991667897376097
1937800210693093944880154046657842803665085489749465821601081073488803416365562328288874281047352143
8522657802640122812472274276284251036543088939277309744946828029516575809109373758180254396617833619
1016
Bytes (1024 bytes): 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
 00 00 00 00 00 3a 7d d2 35 d8 9d a8 e1 49 82 1a 00 6b 24 62 95 6b c0 f1 c2 15 4e 3d 65 a5 e7 40 60
0e 93 48 a1 d4 c5 a7 f8 8b 0c 29 83 67 3c 91 6a d0 2d 79 cd 9b ee 45 99 42 4f 0e b5 45 4c 24 d0 64 0
a 89 13 48 3c 73 11 a9 70 ea f4 6b 94 22 e6 0a 99 19 54 b4 24 e1 0d 2b bf b7 f5 1a 3a 4a 6d e3 ae 3d
 34 e2 3c e2 3e 82 36 fd d2 ec 43 af b3 86 74 c3 cd b7 a2 78 ec 55 64 27 29 8e 34 2c b4 8a ee af 76
65 09 00 0d 19 e7 9c f3 dc a6 bd 2d 6f 91 74 a4 be ff 95 58 3f 26 04 94 14 d4 65 8a c2 32 f1 e9 50 2
d cd a9 ec ff ff c1 b9 ba ea 2f ce 88 e8 16 bc 5a 72 cd 8e e5 57 5a cd 79 07 2f 4f 4c de 48 a7 90 52
 c3 9b aa d1 71 76 80 f5 96 a6 91 22 26 1d 22 71 4a 9b d8 16 9d 5d 05 f4 8c 53 b9 6f 5e 6b e3 67 b9
b7 96 0a 10 cb e8 75 e9 36 f0 41 28 1d ad 15 54 f5 c1 6b 23 12 1f e1 32 c7 45 d2 d2 17 a2 f4 97 84 b
4 a2 60 32 07 97 cd 65 c1 f5 2c c4 8d ed 56 11 4d 86 8c 12 eb ec e7 76 4f 51 f1 30 bc 68 a1 51 46 ed
 88 10 72 2d 03 48 21 a1 f6 c8 6f 6a c7 39 ee b0 4d 67 49 a0 09 7c 62 16 24 65 23 ff 5a 96 0f 19 33
a3 41 71 61 48 30 41 24 ca 4d c7 bf 3b 55 91 73 65 7a be 59 be ce c5 2b 5a 4c 87 31 03 f5 93 15 72 4
f 41 1b e8 bc 86 2e b2 35 5b a4 25 2d 9c a4 81 00 da 70 30 3c 9d 3e 24 56 c0 d1 71 29 ad 38 33 3d 2d
 0b 52 20 6e 12 c7 7e cd 10 fe be 2e c5 0f b7 3c 27 4d b1 96 54 19 57 09 a2 7c bf e0 64 02 c6 be 6f
96 60 d7 8d e4 be 20 3c 7f 98 a8 22 cb 4a e0 67 e0 34 e4 4d 64 6b 7c 40 55 d1 29 2d f5 ac 94 9d 17 0
9 8a d2 b2 5d e2 3f 4a 7c 44 9b 8b c4 7c bc 9d 4c d7 f2 14 0c ff 0f 8a 81 7e 24 24 5e bb 8c a8 4e b3
 ef 6d 73 94 0f a0 bd 86 bf 39 dd 9f 9a e0 48 55 11 5a 9c 42 00 fa 70 00 9d e6 f8 7b dc 18 9e 3d 4a
f7 c8 a1 11 3e 36 bf 98 f3 5c db fe f3 d2 c6 c3 72 bc c9 70 78 04 ea de a0 77 6f a3 66 6d 26 de 94 e
9 25 fc 7e 51 b0 c6 21 46 f9 0f e8 34 69 51 1c 23 20 e4 c5 a8 23 8b 50 f3 95 98 3a 4d 12 c8 ab f7 b8
 87 55 15 c7 ed 2e c7 31 67 ca e3 08 75 ac 1b f8 53 99 cd 0b e5 bb e7 b1 b9 9d 81 b8 f7 83 28 c4 88
c9 a2 8b d1 83 9b c3 49 33 eb 64 60 68 2b 2e ea e7 c8 f1 3f 84 3e 1f 69 4e a6 3e f6 e3 df 8d ff ee 8
6 75 52 7c ae ed 3e c7 13 40 f3 a4 66 56 0d 8c ec 4a b4 c2 10 4e d6 41 3d a2 8c b5 53 4d b9 b7 28 23
 7c e9 ce 01 c6 15 1a e8 3a af aa 81 04 5d 99 2d 47 34 b3 80 3c 4f 1d b4 53 4a 37 12 1d 2e 79 96 ba
40 36 88 dd 97 dc a8 1a 6a 1e 18 96 62 7d e1 aa cf 22 c6 9b cc 29 03 ff 87 6a 41 ad b3 28 96 62 ef 4
e a5 a8 32 b8 08 b2 9e c3 87 4c 4f bf 1c ff dc c1 a2 4c 1a 96 99 c4 19 2a f2 27 39 3d 88 b6 ff 78 6f
 9f ae cb 2a 16 d6 50 90 6b 90 3f 3b ca d3 b5 bd e9 c1 3e cb d3 9d d3 7b 9d 18 7f b1 b6 46 76 e0 04
e0 a0 9e d3 bd d5 cb 65 76 ec fd db 5e c3 e5 7c c5 03 55 6d 76 92 40 ce 95 aa b7 f5 48 ec ac 78 70 d
0 23 bc c4 f7 a9 c4 ae 5a 27 86 00 2c a9 71 c9 bd 55 75 40 57 8d d5 86 6c ce 2e d4 66 60 3d e5 4a 49
 08 a7 58 b6 10 74 a2 d4 85 05 b9 21 3f 7e 98 ee 2b 70 25 a3 84 29 6f 99 35 b5 7b e4 0e 9f a8 98 e6
e7 8b 42 55 5c e9 96 9f 83 a8 7b 3a 83 58 98 dd f4 77 88 2f 43 ab 6e 72 c2 d4 7f fd 3e e9 13 be c6 9
7 b3 cb e5 16 66 fc 2d c6 04 a0 f3 ad 38 83 8c a6 b8 bf 22 ae 6a 83 d1 ba 12 2d 9e f6 32 28
Recreated from Integer: SCCYZPFOBRIQGZYGROFQRDKEETSGVENAIHXMEKXDVFLJKOGMXDZJNJFADLACXIZIDDKOLYZHPACE
XGMSHNFVTKFUTHEHBLSOKIYIVIMJJEKOAKPDLNWWQXTLPSICKUFVZSHZPVAOESCKFQUETKVTVTRGYHDQBQXQVUSTFJEXFDTTHKFP
MBQHJWNUJGUEZCHZLDVQAUJROMZCXOEQWQUGDMEMNXVHIODOBWXYBQRYMGKNTMBMEWNCSTCHFKPUAAJMOIYEPZUWPSIILEESQOZP
KPFJFCETIPDPHRIGIMZPSFVDHWWUHXGHCDEKZJVDOJBQQQLHHXIKJYLCQCVYUWMSUQCJNFPLSFZUZHFPBOAGHAHCXDHXDHKTZCCI
UQPIYQZLZJCSBDCWVSDKWSXPGHDFKISRFEIVBSNLPTHOWUFEIWIFLYULHPBYCKGQILXLTCNNCHRQWBLOEVZZPQSPXBJSQJKFRAXC
WRJEFAINTWDNHTVPDEEYVLWLJITIPJGRIRRK8162706247727809194516824359923413916082951313973998830079786529
7880216629651585263702264572046211260947032516809909181309479444701536646747955594448892351918458848
8099642896816431667893486930459526964785526758583690311025802834027520981355416779466707116813405788
8432689140996240442003415785559987925681989530013371469488885741860953594619163757789113582300420627
0868626534540172380722954238544925015006976927151417989468093082073153424195719398541064699024700241
6520434643532898310932499365952733444103887167167272094077549257959439954040898922812591867413227295
9973719943909695381282774549913671042232930927119139630896356148479827188675968104778477020706056059
3192748501140468374285608384141431779592082948336048838832095149463939309223016467423582482751871522
8438-84969615826702367528886267075424080709640399051738180969188621916878381383129538870924446293079
4386995622425932596930454614804928513469215604102171973068488981669181831876317758709217585355360712
5941022614702145644631117676761967098893284094057673300018784839940731207690376918504870869299600460
6817055360589218886336550418029951548475411383439087316611488421300574688522414066629344304762990175
1287844189311219273979098566511778485090705027459047748793632372560827592710520013750243223642632607
4159712946985220380202858123992427189135331287948774304896367026066128541691198470939524145155084496
9423465989543787186709974824616588038257691687448528933691280710475346671257695685991667897376097193
7800210693093944880154046657842803665085489749465821601081073488803416365562328288874281047352143852
2657802640122812472274276284251036543088939277309744946828029516575809109373758180254396617833619101
6
Verification successful!
```

Haskell：

```haskell
-- file: GSCU.hs
-- build with: cabal build
-- 需要megaparsec和bytestring这两个依赖
{-# LANGUAGE OverloadedStrings #-}

import Data.Void
import Text.Megaparsec
import Text.Megaparsec.Char
import qualified Text.Megaparsec.Char.Lexer as L
import Data.Char (ord, chr, toUpper)
import Data.List (unfoldr, reverse)
import Numeric (showHex)
import Data.Bits (shiftR)
import qualified Data.ByteString as B
import Text.Printf (printf)
import Data.Word (Word8)

-- 1. 数据类型定义
data GSCUIdentifier =
    Club { club :: String }
  | Team { club :: String, team :: String }
  | Stronghold { club :: String, team :: String, stronghold :: String }
  | Researcher { club :: String, team :: String, stronghold :: String, researcher :: String }
  deriving (Eq)

instance Show GSCUIdentifier where
    show (Club c) = c
    show (Team c t) = c ++ t
    show (Stronghold c t s) = c ++ t ++ "-" ++ s
    show (Researcher c t s r) = c ++ t ++ "-" ++ s ++ r

data GSCUConfig = GSCUConfig {
    teamDigits :: Int,
    strongholdDigits :: Int,
    researcherDigits :: Int
} deriving (Show)

defaultConfig :: GSCUConfig
defaultConfig = GSCUConfig 4 3 5

-- 纯整数快速幂
power :: Integer -> Int -> Integer
power base exp
    | exp < 0   = error "power: negative exponent"
    | exp == 0  = 1
    | otherwise = go base exp
    where
      go b e | e == 1 = b
             | even e = let p = go b (e `div` 2) in p * p
             | odd e  = b * let p = go b ((e - 1) `div` 2) in p * p

-- 2. 核心转换逻辑
codenameToInt :: String -> Integer
codenameToInt cname
    | len < 2   = error "Codename must have at least 2 letters."
    | otherwise = foldl (\acc c -> acc * 26 + fromIntegral (ord (toUpper c) - ord 'A')) (0 :: Integer) cname + offset
  where
    len = length cname
    offset = if len > 2 then (power 26 len - 676) `div` 25 else 0

intToCodename :: Integer -> String
intToCodename n
    | n < 0     = error "Integer cannot be negative."
    | otherwise = intToLetters len (n - offset)
  where
    len = if n < 676 then 2 else findCodenameLen n 2
    offset = if len > 2 then (power 26 len - 676) `div` 25 else 0
    findCodenameLen num currentLen =
        let nextOffsetStart = (power 26 (currentLen + 1) - 676) `div` 25
        in if num < nextOffsetStart then currentLen else findCodenameLen num (currentLen + 1)
    
    intToLetters :: Int -> Integer -> String
    intToLetters l num =
      let toChars x | x < 0 = error "Negative number in toChars"
                    | x == 0 = ""
                    | otherwise = toChars (x `div` 26) ++ [chr (fromIntegral (x `mod` 26) + ord 'A')]
          s = toChars num
      in replicate (l - length s) 'A' ++ s


toInteger' :: GSCUIdentifier -> GSCUConfig -> Integer
toInteger' (Club c) _ = codenameToInt c
toInteger' (Team c t) cfg = (codenameToInt c * (power 10 (teamDigits cfg))) + read t
toInteger' (Stronghold c t s) cfg = ((codenameToInt c * (power 10 (teamDigits cfg))) + read t) * (power 10 (strongholdDigits cfg)) + read s
toInteger' (Researcher c t s r) cfg = (((codenameToInt c * (power 10 (teamDigits cfg))) + read t) * (power 10 (strongholdDigits cfg)) + read s) * (power 10 (researcherDigits cfg)) + read r

fromInteger' :: Integer -> String -> GSCUConfig -> GSCUIdentifier
fromInteger' num "club" _ = Club (intToCodename num)
fromInteger' num level cfg =
    let (researcherPart, num1) = extractPart num (researcherDigits cfg) (level == "researcher")
        (strongholdPart, num2) = extractPart num1 (strongholdDigits cfg) (level `elem` ["researcher", "stronghold"])
        (teamPart, clubNum)    = extractPart num2 (teamDigits cfg) (level `elem` ["researcher", "stronghold", "team"])
        clubPart = intToCodename clubNum
    in case level of
        "team"       -> Team clubPart teamPart
        "stronghold" -> Stronghold clubPart teamPart strongholdPart
        "researcher" -> Researcher clubPart teamPart strongholdPart researcherPart
        _            -> error $ "Invalid level for fromInteger': " ++ level
  where
    extractPart n digits shouldExtract =
        if shouldExtract
            then let (q, r) = n `divMod` (power 10 digits) in (printf ("%0" ++ show digits ++ "d") r, q)
            else ("", n)

integerToBytes :: Integer -> B.ByteString
integerToBytes 0 = B.singleton 0
integerToBytes i | i < 0 = error "integerToBytes on negative number"
                 | otherwise = B.pack $ reverse $ unfoldr (\x -> if x == 0 then Nothing else Just (fromInteger x, x `shiftR` 8)) i

-- 3. Megaparsec 解析器
type Parser = Parsec Void String

fixedDigits :: Int -> Parser String
fixedDigits n = count n digitChar <?> (show n ++ " digits")

parseIdentifier :: GSCUConfig -> Parser GSCUIdentifier
parseIdentifier cfg = do
    c <- some letterChar <?> "club codename"
    (try (parseResearcher c) <|> try (parseStronghold c) <|> try (parseTeam c) <|> pure (Club c)) <* eof
  where
    parseTeam c = Team c <$> fixedDigits (teamDigits cfg)
    parseStronghold c = Stronghold c <$> fixedDigits (teamDigits cfg) <* char '-' <*> fixedDigits (strongholdDigits cfg)
    parseResearcher c = Researcher c <$> fixedDigits (teamDigits cfg) <* char '-' <*> fixedDigits (strongholdDigits cfg) <*> fixedDigits (researcherDigits cfg)

-- 4. 统一的测试函数
runTest :: String -> Int -> GSCUConfig -> IO ()
runTest codeStr extendBytes cfg = do
    putStrLn $ "--- Testing: " ++ codeStr ++ " ---"
    case runParser (parseIdentifier cfg) "" codeStr of
        Left bundle -> putStr (errorBundlePretty bundle)
        Right identifier -> do
            let intVal = toInteger' identifier cfg
            let level = case identifier of
                            Club {} -> "club"; Team {} -> "team"; Stronghold {} -> "stronghold"; Researcher {} -> "researcher"
            let recreatedId = fromInteger' intVal level cfg
            printf "Parsed OK. Level: %s\n" level
            putStrLn $ "Integer value: " ++ show intVal
            let bytes = integerToBytes intVal
            let paddedBytes = B.replicate (extendBytes - B.length bytes) 0 `B.append` bytes
            putStrLn $ "Bytes (" ++ show extendBytes ++ " bytes): " ++ formatBytes paddedBytes
            putStrLn $ "Recreated from Integer: " ++ show recreatedId
            if show identifier == show recreatedId
                then putStrLn "Verification successful!"
                else putStrLn "Verification FAILED!"
    putStrLn ""
  where
    formatBytes = concatMap (printf "%02x") . B.unpack

-- 5. 主函数入口
main :: IO ()
main = do
    runTest "AL" 2 defaultConfig
    runTest "LFN" 2 defaultConfig
    runTest "LFN2018" 4 defaultConfig
    runTest "LFN2017" 4 defaultConfig
    runTest "LFN2016" 4 defaultConfig
    runTest "LFN2010" 4 defaultConfig
    runTest "LFN2009" 4 defaultConfig
    runTest "LFN2008" 4 defaultConfig
    runTest "LFM2018" 4 defaultConfig
    runTest "LFN2018-001" 6 defaultConfig
    runTest "LFN2018-00121376" 8 defaultConfig

    let extConfig = GSCUConfig 5 4 6
    runTest "ABCD12345-1234123456" 10 extConfig
    runTest "ABCD12345-0079009999" 10 extConfig
    runTest "ABCD12345-0073004250" 10 extConfig
    runTest "ABCD12345-1234123456" 12 extConfig
    runTest "ABCD12345-0079009999" 12 extConfig
    runTest "ABCD12345-0073004250" 12 extConfig

    -- 极端测试用例1：32字母代号、128位队编号、32位据点编号、160位研究员编号
    let extremeConfig1 = GSCUConfig 128 32 160
    runTest "OGSFMTZUYPDWEUWXVVXKNJRYNKLUBBTW78277734991523235117860819759786988365517279210025614105326173650938950916302232452494230432958470080777285191609328915906043578-776261560186981234481971652350272969792974632663687899104508087897537776077509140745424387085295119775745153335627494550643972976201292294196255472025200935787200791851143729323167242337059890" 160 extremeConfig1

    runTest "DRRXOVDWUSBOSSJRPKDJDAESHBGLYZXF44279291197287017989197737530221054754281591586161309649455260922205242752540347239070056122383958662760500971883425417489971708-929068242189337992176223731734560165649707397842823033520967641749386131644397754241429111292220932814616795545657306326072542986333657756618096778539458485186964449754475259887722050899685392" 160 extremeConfig1

    runTest "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA44279291197287017989197737530221054754281591586161309649455260922205242752540347239070056122383958662760500971883425417489971708-929068242189337992176223731734560165649707397842823033520967641749386131644397754241429111292220932814616795545657306326072542986333657756618096778539458485186964449754475259887722050899685392" 160 extremeConfig1

    runTest "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAABA44279291197287017989197737530221054754281591586161309649455260922205242752540347239070056122383958662760500971883425417489971708-929068242189337992176223731734560165649707397842823033520967641749386131644397754241429111292220932814616795545657306326072542986333657756618096778539458485186964449754475259887722050899685392" 160 extremeConfig1

    runTest "BAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA44279291197287017989197737530221054754281591586161309649455260922205242752540347239070056122383958662760500971883425417489971708-929068242189337992176223731734560165649707397842823033520967641749386131644397754241429111292220932814616795545657306326072542986333657756618096778539458485186964449754475259887722050899685392" 160 extremeConfig1

    -- 极端测试用例2：512字母代号、768位队编号、384位据点编号、512位研究员编号
    let extremeConfig2 = GSCUConfig 768 384 512
    runTest "SCCYZPFOBRIQGZYGROFQRDKEETSGVENAIHXMEKXDVFLJKOGMXDZJNJFADLACXIZIDDKOLYZHPACEXGMSHNFVTKFUTHEHBLSOKIYIVIMJJEKOAKPDLNWWQXTLPSICKUFVZSHZPVAOESCKFQUETKVTVTRGYHDQBQXQVUSTFJEXFDTTHKFPMBQHJWNUJGUEZCHZLDVQAUJROMZCXOEQWQUGDMEMNXVHIODOBWXYBQRYMGKNTMBMEWNCSTCHFKPUAAJMOIYEPZUWPSIILEESQOZPKPFJFCETIPDPHRIGIMZPSFVDHWWUHXGHCDEKZJVDOJBQQQLHHXIKJYLCQCVYUWMSUQCJNFPLSFZUZHFPBOAGHAHCXDHXDHKTZCCIUQPIYQZLZJCSBDCWVSDKWSXPGHDFKISRFEIVBSNLPTHOWUFEIWIFLYULHPBYCKGQILXLTCNNCHRQWBLOEVZZPQSPXBJSQJKFRAXCWRJEFAINTWDNHTVPDEEYVLWLJITIPJGRIRRK816270624772780919451682435992341391608295131397399883007978652978802166296515852637022645720462112609470325168099091813094794447015366467479555944488923519184588488099642896816431667893486930459526964785526758583690311025802834027520981355416779466707116813405788843268914099624044200341578555998792568198953001337146948888574186095359461916375778911358230042062708686265345401723807229542385449250150069769271514179894680930820731534241957193985410646990247002416520434643532898310932499365952733444103887167167272094077549257959439954040898922812591867413227295997371994390969538128277454991367104223293092711913963089635614847982718867596810477847702070605605931927485011404683742856083841414317795920829483360488388320951494639393092230164674235824827518715228438-84969615826702367528886267075424080709640399051738180969188621916878381383129538870924446293079438699562242593259693045461480492851346921560410217197306848898166918183187631775870921758535536071259410226147021456446311176767619670988932840940576733000187848399407312076903769185048708692996004606817055360589218886336550418029951548475411383439087316611488421300574688522414066629344304762990175128784418931121927397909856651177848509070502745904774879363237256082759271052001375024322364263260741597129469852203802028581239924271891353312879487743048963670260661285416911984709395241451550844969423465989543787186709974824616588038257691687448528933691280710475346671257695685991667897376097193780021069309394488015404665784280366508548974946582160108107348880341636556232828887428104735214385226578026401228124722742762842510365430889392773097449468280295165758091093737581802543966178336191016" 1024 extremeConfig2
```

运行结果：

```plaintext
--- Testing: AL ---
Parsed OK. Level: club
Integer value: 11
Bytes (2 bytes): 000b
Recreated from Integer: AL
Verification successful!
--- Testing: LFN ---
Parsed OK. Level: club
Integer value: 8255
Bytes (2 bytes): 203f
Recreated from Integer: LFN
Verification successful!
--- Testing: LFN2018 ---
Parsed OK. Level: team
Integer value: 82552018
Bytes (4 bytes): 04eba4d2
Recreated from Integer: LFN2018
Verification successful!
--- Testing: LFN2017 ---
Parsed OK. Level: team
Integer value: 82552017
Bytes (4 bytes): 04eba4d1
Recreated from Integer: LFN2017
Verification successful!
--- Testing: LFN2016 ---
Parsed OK. Level: team
Integer value: 82552016
Bytes (4 bytes): 04eba4d0
Recreated from Integer: LFN2016
Verification successful!
--- Testing: LFN2010 ---
Parsed OK. Level: team
Integer value: 82552010
Bytes (4 bytes): 04eba4ca
Recreated from Integer: LFN2010
Verification successful!
--- Testing: LFN2009 ---
Parsed OK. Level: team
Integer value: 82552009
Bytes (4 bytes): 04eba4c9
Recreated from Integer: LFN2009
Verification successful!
--- Testing: LFN2008 ---
Parsed OK. Level: team
Integer value: 82552008
Bytes (4 bytes): 04eba4c8
Recreated from Integer: LFN2008
Verification successful!
--- Testing: LFM2018 ---
Parsed OK. Level: team
Integer value: 82542018
Bytes (4 bytes): 04eb7dc2
Recreated from Integer: LFM2018
Verification successful!
--- Testing: LFN2018-001 ---
Parsed OK. Level: stronghold
Integer value: 82552018001
Bytes (6 bytes): 0013387bd451
Recreated from Integer: LFN2018-001
Verification successful!
--- Testing: LFN2018-00121376 ---
Parsed OK. Level: researcher
Integer value: 8255201800121376
Bytes (8 bytes): 001d540ff2d86c20
Recreated from Integer: LFN2018-00121376
Verification successful!
--- Testing: ABCD12345-1234123456 ---
Parsed OK. Level: researcher
Integer value: 18983123451234123456
Bytes (10 bytes): 000107719a33b6327ec0
Recreated from Integer: ABCD12345-1234123456
Verification successful!
--- Testing: ABCD12345-0079009999 ---
Parsed OK. Level: researcher
Integer value: 18983123450079009999
Bytes (10 bytes): 000107719a337158dccf
Recreated from Integer: ABCD12345-0079009999
Verification successful!
--- Testing: ABCD12345-0073004250 ---
Parsed OK. Level: researcher
Integer value: 18983123450073004250
Bytes (10 bytes): 000107719a3370fd38da
Recreated from Integer: ABCD12345-0073004250
Verification successful!
--- Testing: ABCD12345-1234123456 ---
Parsed OK. Level: researcher
Integer value: 18983123451234123456
Bytes (12 bytes): 0000000107719a33b6327ec0
Recreated from Integer: ABCD12345-1234123456
Verification successful!
--- Testing: ABCD12345-0079009999 ---
Parsed OK. Level: researcher
Integer value: 18983123450079009999
Bytes (12 bytes): 0000000107719a337158dccf
Recreated from Integer: ABCD12345-0079009999
Verification successful!
--- Testing: ABCD12345-0073004250 ---
Parsed OK. Level: researcher
Integer value: 18983123450073004250
Bytes (12 bytes): 0000000107719a3370fd38da
Recreated from Integer: ABCD12345-0073004250
Verification successful!
--- Testing: OGSFMTZUYPDWEUWXVVXKNJRYNKLUBBTW78277734991523235117860819759786988365517279210025614105326173650938950916302232452494230432958470080777285191609328915906043578-776261560186981234481971652350272969792974632663687899104508087897537776077509140745424387085295119775745153335627494550643972976201292294196255472025200935787200791851143729323167242337059890 ---
Parsed OK. Level: researcher
Integer value: 111892294247529835148296226357258524292822609278277734991523235117860819759786988365517279210025614105326173650938950916302232452494230432958470080777285191609328915906043578776261560186981234481971652350272969792974632663687899104508087897537776077509140745424387085295119775745153335627494550643972976201292294196255472025200935787200791851143729323167242337059890
Bytes (160 bytes): 000000000000000019626258a5e36db6abc3e25444b928ef983a63224887540508ff04f25370d1ac86261c1a11bc77abf66c246462300a5a34edcfd417ef646300305ae0c275bb1ef7ada777fdc33bd33314cbbe99477cf4075f1128db53c989fd6cc35c8c760f27116f25964ecca346795cbfa22b42e93ac3459d94ffe0f919809fd2731c0e53f82d411717693955f02e2aebbe0acef39bb5db76531caf0432
Recreated from Integer: OGSFMTZUYPDWEUWXVVXKNJRYNKLUBBTW78277734991523235117860819759786988365517279210025614105326173650938950916302232452494230432958470080777285191609328915906043578-776261560186981234481971652350272969792974632663687899104508087897537776077509140745424387085295119775745153335627494550643972976201292294196255472025200935787200791851143729323167242337059890
Verification successful!
--- Testing: DRRXOVDWUSBOSSJRPKDJDAESHBGLYZXF44279291197287017989197737530221054754281591586161309649455260922205242752540347239070056122383958662760500971883425417489971708-929068242189337992176223731734560165649707397842823033520967641749386131644397754241429111292220932814616795545657306326072542986333657756618096778539458485186964449754475259887722050899685392 ---
Parsed OK. Level: researcher
Integer value: 34526027956558810660405438139965204833484877944279291197287017989197737530221054754281591586161309649455260922205242752540347239070056122383958662760500971883425417489971708929068242189337992176223731734560165649707397842823033520967641749386131644397754241429111292220932814616795545657306326072542986333657756618096778539458485186964449754475259887722050899685392
Bytes (160 bytes): 000000000000000007d52c418b0996696cb8ef7df0fee268a90e85a05d7bd4cf16f0bbe6754ca424c60a4336690c279961c05c3c5dab6dd6da85ee7ea38ba1d68fb6b71f84410495a2ace47486d069eaef0c58927e2aa89a72fd4310498a313b0e5ad487001cea5b979875d2ab9b4d1f3f5e8a66b09a2d424fe7102ffc2e1badbaed37e7dec338aa6e81a44b682de9771514056710b04fd30a7308cb06343010
Recreated from Integer: DRRXOVDWUSBOSSJRPKDJDAESHBGLYZXF44279291197287017989197737530221054754281591586161309649455260922205242752540347239070056122383958662760500971883425417489971708-929068242189337992176223731734560165649707397842823033520967641749386131644397754241429111292220932814616795545657306326072542986333657756618096778539458485186964449754475259887722050899685392
Verification successful!
--- Testing: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA44279291197287017989197737530221054754281591586161309649455260922205242752540347239070056122383958662760500971883425417489971708-929068242189337992176223731734560165649707397842823033520967641749386131644397754241429111292220932814616795545657306326072542986333657756618096778539458485186964449754475259887722050899685392 ---
Parsed OK. Level: researcher
Integer value: 7606889829073952965675311264081586992084678044279291197287017989197737530221054754281591586161309649455260922205242752540347239070056122383958662760500971883425417489971708929068242189337992176223731734560165649707397842823033520967641749386131644397754241429111292220932814616795545657306326072542986333657756618096778539458485186964449754475259887722050899685392
Bytes (160 bytes): 000000000000000001b9c950a8f3aa5409239a42a920d5599825b3b7f76bba2f6e216d4d6eef9bcf07af12af3fde7b109a2eb7d553cc13c8ebee61f95b97b33eea9629ccb816c7a9e7837884ebdaff8e7cc6444f7ec85ac7a8e19bb7e2e51021d7910f6f18c6825cf2f9261ba23da49a71268dcbee5c78f34fe7102ffc2e1badbaed37e7dec338aa6e81a44b682de9771514056710b04fd30a7308cb06343010
Recreated from Integer: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA44279291197287017989197737530221054754281591586161309649455260922205242752540347239070056122383958662760500971883425417489971708-929068242189337992176223731734560165649707397842823033520967641749386131644397754241429111292220932814616795545657306326072542986333657756618096778539458485186964449754475259887722050899685392
Verification successful!
--- Testing: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAABA44279291197287017989197737530221054754281591586161309649455260922205242752540347239070056122383958662760500971883425417489971708-929068242189337992176223731734560165649707397842823033520967641749386131644397754241429111292220932814616795545657306326072542986333657756618096778539458485186964449754475259887722050899685392 ---
Parsed OK. Level: researcher
Integer value: 7606889829073952965675311264081586992084680644279291197287017989197737530221054754281591586161309649455260922205242752540347239070056122383958662760500971883425417489971708929068242189337992176223731734560165649707397842823033520967641749386131644397754241429111292220932814616795545657306326072542986333657756618096778539458485186964449754475259887722050899685392
Bytes (160 bytes): 000000000000000001b9c950a8f3aa5409239a42a920d5599825c0df6435fcd83d3b66b9cb1ee9e49a11961e3ec07f9cdc3ad1dc694000b63fd142c0140739423eb295652274bb73553be8e7f3c51c797254038f566beebbf14c863fcb4bbd5a21602022e7e30d88b51cfdb758813b26bd90384e1e44370d4fe7102ffc2e1badbaed37e7dec338aa6e81a44b682de9771514056710b04fd30a7308cb06343010
Recreated from Integer: AAAAAAAAAAAAAAAAAAAAAAAAAAAAAABA44279291197287017989197737530221054754281591586161309649455260922205242752540347239070056122383958662760500971883425417489971708-929068242189337992176223731734560165649707397842823033520967641749386131644397754241429111292220932814616795545657306326072542986333657756618096778539458485186964449754475259887722050899685392
Verification successful!
--- Testing: BAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA44279291197287017989197737530221054754281591586161309649455260922205242752540347239070056122383958662760500971883425417489971708-929068242189337992176223731734560165649707397842823033520967641749386131644397754241429111292220932814616795545657306326072542986333657756618096778539458485186964449754475259887722050899685392 ---
Parsed OK. Level: researcher
Integer value: 14921206972414292355747725941083112946012255644279291197287017989197737530221054754281591586161309649455260922205242752540347239070056122383958662760500971883425417489971708929068242189337992176223731734560165649707397842823033520967641749386131644397754241429111292220932814616795545657306326072542986333657756618096778539458485186964449754475259887722050899685392
Bytes (160 bytes): 0000000000000000036294bbc18f3091259e7382c1e7c9e0f9364fecd4d7d1dcb646bfd7d5e7cd41408895acfcded9ab859507e7967a7e38d98efe0c2e9aa6d21e5e951f0e13b6b4882f74d0c33ba1337fe66a3be46d749f578d885671921387147f7dd2568c1004f8107e310b8f10d5dc3da5ee6e5c78f34fe7102ffc2e1badbaed37e7dec338aa6e81a44b682de9771514056710b04fd30a7308cb06343010
Recreated from Integer: BAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA44279291197287017989197737530221054754281591586161309649455260922205242752540347239070056122383958662760500971883425417489971708-929068242189337992176223731734560165649707397842823033520967641749386131644397754241429111292220932814616795545657306326072542986333657756618096778539458485186964449754475259887722050899685392
Verification successful!
--- Testing: SCCYZPFOBRIQGZYGROFQRDKEETSGVENAIHXMEKXDVFLJKOGMXDZJNJFADLACXIZIDDKOLYZHPACEXGMSHNFVTKFUTHEHBLSOKIYIVIMJJEKOAKPDLNWWQXTLPSICKUFVZSHZPVAOESCKFQUETKVTVTRGYHDQBQXQVUSTFJEXFDTTHKFPMBQHJWNUJGUEZCHZLDVQAUJROMZCXOEQWQUGDMEMNXVHIODOBWXYBQRYMGKNTMBMEWNCSTCHFKPUAAJMOIYEPZUWPSIILEESQOZPKPFJFCETIPDPHRIGIMZPSFVDHWWUHXGHCDEKZJVDOJBQQQLHHXIKJYLCQCVYUWMSUQCJNFPLSFZUZHFPBOAGHAHCXDHXDHKTZCCIUQPIYQZLZJCSBDCWVSDKWSXPGHDFKISRFEIVBSNLPTHOWUFEIWIFLYULHPBYCKGQILXLTCNNCHRQWBLOEVZZPQSPXBJSQJKFRAXCWRJEFAINTWDNHTVPDEEYVLWLJITIPJGRIRRK816270624772780919451682435992341391608295131397399883007978652978802166296515852637022645720462112609470325168099091813094794447015366467479555944488923519184588488099642896816431667893486930459526964785526758583690311025802834027520981355416779466707116813405788843268914099624044200341578555998792568198953001337146948888574186095359461916375778911358230042062708686265345401723807229542385449250150069769271514179894680930820731534241957193985410646990247002416520434643532898310932499365952733444103887167167272094077549257959439954040898922812591867413227295997371994390969538128277454991367104223293092711913963089635614847982718867596810477847702070605605931927485011404683742856083841414317795920829483360488388320951494639393092230164674235824827518715228438-84969615826702367528886267075424080709640399051738180969188621916878381383129538870924446293079438699562242593259693045461480492851346921560410217197306848898166918183187631775870921758535536071259410226147021456446311176767619670988932840940576733000187848399407312076903769185048708692996004606817055360589218886336550418029951548475411383439087316611488421300574688522414066629344304762990175128784418931121927397909856651177848509070502745904774879363237256082759271052001375024322364263260741597129469852203802028581239924271891353312879487743048963670260661285416911984709395241451550844969423465989543787186709974824616588038257691687448528933691280710475346671257695685991667897376097193780021069309394488015404665784280366508548974946582160108107348880341636556232828887428104735214385226578026401228124722742762842510365430889392773097449468280295165758091093737581802543966178336191016 ---
Parsed OK. Level: researcher
Integer value: 2152277669617411707527228506614657545274393961758934268513055146236306579787175928933464170907097955045862799163382684421589504391164810942926125557900482075362912261217240809228824261730817583869078631679141566843961254883676525850762842502013998456664917471100217196992099561071666896346191437261386798091961754189179798735314070100576986236015035873416618930439572051430551333991040870137490498660500664240392657465560601799691246212434896293069138346541294699935924403613118824501150913623998010751232836907998566617621289416694493885642882171867709291028717247008241661145138514810643643878507552277712248132227301272186642432098517999908772152490813549212592841168148474225432519677385677454495563749122790493670259384481627062477278091945168243599234139160829513139739988300797865297880216629651585263702264572046211260947032516809909181309479444701536646747955594448892351918458848809964289681643166789348693045952696478552675858369031102580283402752098135541677946670711681340578884326891409962404420034157855599879256819895300133714694888857418609535946191637577891135823004206270868626534540172380722954238544925015006976927151417989468093082073153424195719398541064699024700241652043464353289831093249936595273344410388716716727209407754925795943995404089892281259186741322729599737199439096953812827745499136710422329309271191396308963561484798271886759681047784770207060560593192748501140468374285608384141431779592082948336048838832095149463939309223016467423582482751871522843884969615826702367528886267075424080709640399051738180969188621916878381383129538870924446293079438699562242593259693045461480492851346921560410217197306848898166918183187631775870921758535536071259410226147021456446311176767619670988932840940576733000187848399407312076903769185048708692996004606817055360589218886336550418029951548475411383439087316611488421300574688522414066629344304762990175128784418931121927397909856651177848509070502745904774879363237256082759271052001375024322364263260741597129469852203802028581239924271891353312879487743048963670260661285416911984709395241451550844969423465989543787186709974824616588038257691687448528933691280710475346671257695685991667897376097193780021069309394488015404665784280366508548974946582160108107348880341636556232828887428104735214385226578026401228124722742762842510365430889392773097449468280295165758091093737581802543966178336191016
Bytes (1024 bytes): 00000000000000000000000000000000000000000000000000000000000000003a7dd235d89da8e149821a006b2462956bc0f1c2154e3d65a5e740600e9348a1d4c5a7f88b0c2983673c916ad02d79cd9bee4599424f0eb5454c24d0640a8913483c7311a970eaf46b9422e60a991954b424e10d2bbfb7f51a3a4a6de3ae3d34e23ce23e8236fdd2ec43afb38674c3cdb7a278ec556427298e342cb48aeeaf766509000d19e79cf3dca6bd2d6f9174a4beff95583f26049414d4658ac232f1e9502dcda9ecffffc1b9baea2fce88e816bc5a72cd8ee5575acd79072f4f4cde48a79052c39baad1717680f596a69122261d22714a9bd8169d5d05f48c53b96f5e6be367b9b7960a10cbe875e936f041281dad1554f5c16b23121fe132c745d2d217a2f49784b4a260320797cd65c1f52cc48ded56114d868c12ebece7764f51f130bc68a15146ed8810722d034821a1f6c86f6ac739eeb04d6749a0097c6216246523ff5a960f1933a341716148304124ca4dc7bf3b559173657abe59becec52b5a4c873103f59315724f411be8bc862eb2355ba4252d9ca48100da70303c9d3e2456c0d17129ad38333d2d0b52206e12c77ecd10febe2ec50fb73c274db19654195709a27cbfe06402c6be6f9660d78de4be203c7f98a822cb4ae067e034e44d646b7c4055d1292df5ac949d17098ad2b25de23f4a7c449b8bc47cbc9d4cd7f2140cff0f8a817e24245ebb8ca84eb3ef6d73940fa0bd86bf39dd9f9ae04855115a9c4200fa70009de6f87bdc189e3d4af7c8a1113e36bf98f35cdbfef3d2c6c372bcc9707804eadea0776fa3666d26de94e925fc7e51b0c62146f90fe83469511c2320e4c5a8238b50f395983a4d12c8abf7b8875515c7ed2ec73167cae30875ac1bf85399cd0be5bbe7b1b99d81b8f78328c488c9a28bd1839bc34933eb6460682b2eeae7c8f13f843e1f694ea63ef6e3df8dffee8675527caeed3ec71340f3a466560d8cec4ab4c2104ed6413da28cb5534db9b728237ce9ce01c6151ae83aafaa81045d992d4734b3803c4f1db4534a37121d2e7996ba403688dd97dca81a6a1e1896627de1aacf22c69bcc2903ff876a41adb3289662ef4ea5a832b808b29ec3874c4fbf1cffdcc1a24c1a9699c4192af227393d88b6ff786f9faecb2a16d650906b903f3bcad3b5bde9c13ecbd39dd37b9d187fb1b64676e004e0a09ed3bdd5cb6576ecfddb5ec3e57cc503556d769240ce95aab7f548ecac7870d023bcc4f7a9c4ae5a2786002ca971c9bd557540578dd5866cce2ed466603de54a4908a758b61074a2d48505b9213f7e98ee2b7025a384296f9935b57be40e9fa898e6e78b42555ce9969f83a87b3a835898ddf477882f43ab6e72c2d47ffd3ee913bec697b3cbe51666fc2dc604a0f3ad38838ca6b8bf22ae6a83d1ba122d9ef63228
Recreated from Integer: SCCYZPFOBRIQGZYGROFQRDKEETSGVENAIHXMEKXDVFLJKOGMXDZJNJFADLACXIZIDDKOLYZHPACEXGMSHNFVTKFUTHEHBLSOKIYIVIMJJEKOAKPDLNWWQXTLPSICKUFVZSHZPVAOESCKFQUETKVTVTRGYHDQBQXQVUSTFJEXFDTTHKFPMBQHJWNUJGUEZCHZLDVQAUJROMZCXOEQWQUGDMEMNXVHIODOBWXYBQRYMGKNTMBMEWNCSTCHFKPUAAJMOIYEPZUWPSIILEESQOZPKPFJFCETIPDPHRIGIMZPSFVDHWWUHXGHCDEKZJVDOJBQQQLHHXIKJYLCQCVYUWMSUQCJNFPLSFZUZHFPBOAGHAHCXDHXDHKTZCCIUQPIYQZLZJCSBDCWVSDKWSXPGHDFKISRFEIVBSNLPTHOWUFEIWIFLYULHPBYCKGQILXLTCNNCHRQWBLOEVZZPQSPXBJSQJKFRAXCWRJEFAINTWDNHTVPDEEYVLWLJITIPJGRIRRK816270624772780919451682435992341391608295131397399883007978652978802166296515852637022645720462112609470325168099091813094794447015366467479555944488923519184588488099642896816431667893486930459526964785526758583690311025802834027520981355416779466707116813405788843268914099624044200341578555998792568198953001337146948888574186095359461916375778911358230042062708686265345401723807229542385449250150069769271514179894680930820731534241957193985410646990247002416520434643532898310932499365952733444103887167167272094077549257959439954040898922812591867413227295997371994390969538128277454991367104223293092711913963089635614847982718867596810477847702070605605931927485011404683742856083841414317795920829483360488388320951494639393092230164674235824827518715228438-84969615826702367528886267075424080709640399051738180969188621916878381383129538870924446293079438699562242593259693045461480492851346921560410217197306848898166918183187631775870921758535536071259410226147021456446311176767619670988932840940576733000187848399407312076903769185048708692996004606817055360589218886336550418029951548475411383439087316611488421300574688522414066629344304762990175128784418931121927397909856651177848509070502745904774879363237256082759271052001375024322364263260741597129469852203802028581239924271891353312879487743048963670260661285416911984709395241451550844969423465989543787186709974824616588038257691687448528933691280710475346671257695685991667897376097193780021069309394488015404665784280366508548974946582160108107348880341636556232828887428104735214385226578026401228124722742762842510365430889392773097449468280295165758091093737581802543966178336191016
Verification successful!
```
